---
layout: post
title: "How SREs Solve Problems at Home (Making my Dumb Fridge Smart)"
---

## Pretext

This is a story partially about laziness but the weird kind of laziness where, instead of doing _the thing_, you decide to do something completely different and more convoluted that ends up taking way more time because you can’t be bothered to do the thing.

In this instance, _the thing_ is fixing my fridge (though I would argue it shouldn’t be my responsibility to do that, I rent and my landlord should have fixed this when it became an issue months ago but oh well).

## My Fridge is Broken

For the record my fridge, being manufactured in 2003, is old enough to vote.

I learned that my fridge was broken by opening the attached freezer on top and looking at a puddle in my ice bin. While this was a useful signal to notice something was wrong with my fridge, I didn’t know what was wrong or how I could fix it. Chatting with my landlord, he mentioned "the previous tenant had a similar issue and whenever it happened, they would cut power to the fridge and wait about 15 minutes and then turn it back on".

For context, the freezer and fridge share a compressor in models like this:
![My fridge/freezer combo unit with the freezer taking up about 1/3 of the unit's space. It sits atop the fridge portion.](/public/images/2021-03-07/fridge.jpg)

Well, at least now I had a remediation step to take, so I gave it a shot and don’t you know, it worked perfectly! Problem solved! …Not really though, unless I want to be checking on my ice bin hourly to know if I needed to power cycle it, I needed another solution.

At this point a normal person might consider trying to repair the fridge or hiring someone to do as much, but it’s been over a month since the fridge technician made his house call and there has been no follow-up while the issue persists. So I hopped on amazon and started to solve the problem the only way I know how: with metrics and automated remediation!

## The Metric

For this problem, there are several metrics we could consider. Initially, my metric was looking in the ice bin to see if it was melting (if it was, this would lead to it all refreezing once the compressor started up again turning my cubes into a giant chunk of ice). While this metric works well for me (a human) it also catches the problem too late (the ice has already begun to melt meaning the freezer is over 32F) and it is harder to measure with a computer (maybe a trained computer vision model could learn to see melting ice differently from non-melting ice but that’s a bit more than I signed up for).

The solution I settled upon was to monitor the temperature inside the freezer with a bluetooth thermometer probe (though another way could be something like monitoring power draw at the wall of the fridge).

I picked up the cheap bluetooth thermometer [here](https://www.amazon.com/gp/product/B08S3CGZ3Q) for around $15 and set to work installing the app and checking it regularly. This let me observe temperatures that the freezer oscillates between during normal operation and what it looks like when the temperature begins to run away:

![Fridge temperature graph showing oscillations between 2.5F and 12F excepting one large spike to around 28F. Oscillations appear to occur every hour or so](/public/images/2021-03-07/fridge_temps.png)

As you can see, the temperature sits somewhere from 2.5F to 12F regularly and occasionally spikes during weird events. Clearly in the image shown the spike was halted by the system I'm about to build in this blog.

So we have a metric and graph viewable in an app. Let’s grab some random FOSS project that can talk to these weird thermometers over bluetooth and slap in in a Raspberry Pi.

## Gathering Data

To interface with the thermometer, instead of using the weird limited app I wondered if someone else had taken a crack at talking to this device directly and it turns out they did! I found a project called [Inkbird](https://github.com/tobievii/inkbird) and [forked it](https://github.com/ericvolp12/inkbird) to twist to my nefarious purposes of making my fridge smart.

In a nutshell, this is a node.js project that utilizes the [noble](https://www.npmjs.com/package/@abandonware/noble) BLE module to talk to Inkbird thermometers. Though this code was written for a previous version, so I had to modify one of the constants to scan BLE devices and filter for the correct device name.

I found the right name through experimenting with a [bluetooth scanning app](https://apps.apple.com/us/app/nrf-connect-bluetooth-app/id1054362403) on my phone and opening and closing the freezer door to see which device jumped in signal when it opened.

I wrote a loop that connects to the thermometer once a minute to avoid killing its battery prematurely. It grabs one datapoint containing a temperature and battery level reading and then disconnects. It then exposes the program state via a JSON API endpoint with Express.

An abbreviated form of the final product can be found below:

```js
const IBS_TH1 = require("./ibs_th1");
const log4js = require("log4js");

const logger_ = log4js.getLogger();
logger_.level = "info";

const device = new IBS_TH1();

let state = {
  connecting: false,
  gotData: false,
  waitCount: 60,
  //...
};

let sensorState = {
  temps: [],
  battery: 0,
  lastUpdated: new Date().toString("en-US"),
};

const callback = (data) => {
  logger_.debug(
    `Current Temp: ${data.temperature.toFixed(2)}°F -` +
      ` Battery: ${data.battery}%`
  );
  sensorState.battery = data.battery;
  if (sensorState.temps.length > 60) {
    sensorState.temps.shift();
  }
  sensorState.temps.push(data.temperature.toFixed(3));
  sensorState.lastUpdated = new Date().toString("en-US");

  state.gotData = true;
  device.unsubscribeRealtimeData();
  logger_.debug("Disconnecting from sensor");
};

const dutyLoop = function () {
  //...
  if (!state.connecting && state.waitCount >= 60) {
    device.subscribeRealtimeData(callback);
    logger_.debug("Connecting to sensor");
    state.connecting = true;
  } else if (!state.connecting) {
    state.waitCount++;
  } else {
    if (state.gotData) {
      state.gotData = false;
      state.waitCount = 0;
      state.connecting = false;
    }
  }
};

logger_.info("Starting duty loop...");
setInterval(dutyLoop, 1000);
```

## Metric Monitoring and Heuristics

This code as written will build a running log of the past 60 minutes of one-per-minute measurements of freezer temperature, and track the current battery life of the sensor.

To enhance it we want to add some kind of heuristic to determine if the freezer is too warm.

We can do that by setting a trigger temperature and then a number of minutes the freezer must be above that temperature to trigger the alarm. We also want to add an alarm cooldown so we don’t trigger once a minute while the freezer temp is rising. Since even if we kick in remediation it can take some time to drop the temps back down.

So we'll add these variables to `state`:

```js
let state = {
    //...
    triggerTemp: 20,
    triggerMinutes: 5,
    lastPowerReset: new Date(0),
    cooldownMin: 60,
    powerCycleWaitTime: 15,
}
```

Nice, now we want to add the heuristic to our loop by grabbing the last `triggerMinutes` worth of measurements from `sensorState.temps` and checking if all of them are over the `triggerTemp`.

```js
const dutyLoop = function(){
  // Evaluate state
  const now = new Date();
  if (now - state.lastPowerReset > state.cooldownMin * 60000) {
    if (
      sensorState.temps
        .slice(-state.triggerMinutes)
        .filter((temp) => temp > state.triggerTemp).length >=
      state.triggerMinutes
    ) {
      logger_.info(
        `Freezer has been over trigger temp of (${state.triggerTemp}F) for at least (${state.triggerMinutes}) minutes.`
      );
      if (now - state.lastPowerReset > state.triggerMinutes * 60000) {
        logger_.info(
          `Resetting fridge power, last reset was at (${state.lastPowerReset.toLocaleString(
            "en-US"
          )})`
        );
        state.lastPowerReset = now;
        // Remediate the problem...
      }
    }
  }
  //...
}
```

Finally, we need to somehow remediate the problem.

## Automated Remediation

We know that powering the fridge off and waiting about 15 minutes and turning it back on seems to remediate the issue. Do I know why that works? No. Do I need to know in order to fix it? Also no. But we can simply automate the power cycling to make this problem fix itself. All I need to do is power off the fridge, wait 15 minutes, and turn it on, but from code.

As a bit of background I wrote this [home automation API in Go](https://github.com/ericvolp12/gohome) a couple months ago which talks to Philips Hue, Belkin WeMo, and Tasmota IOT devices through different protocols and gives me the ability to control them at will and with automation. The interesting bits for this project are the `SetStateHandler` which can be found [here](https://github.com/ericvolp12/gohome/blob/master/cmd/gohome/main.go#L141).

This code allows me to set the power state of a given [Tasmota](https://tasmota.github.io/docs/) outlet connected to a MQTT broker in my apartment. I hacked up some old [Gosund ESP8266 based WiFi Smart Plugs](https://www.amazon.com/gp/product/B079MFTYMV) and installed Tasmota on them with the aid of [TuyaConvert](https://github.com/ct-Open-Source/tuya-convert) which allows you to OverTheAir upgrade some off-the-shelf IOT devices. It is worth knowing that some of the recent Gosund units I've bought from Amazon have updated firmware that is harder to flash and until recently I wasn’t able to update them until the TuyaConvert project managed to crack some of their newer security features for OTA updates.

So we have an API in my home that allows me to talk to an outlet and make it turn on and off, so the last thing to do is integrate it into the temperature tracking code.

We'll use Axios to make a JSON POST request to the API as required.

```js
const axios = require("axios");

const setFridgeState = function (desiredState) {
  axios
    .post(process.env.OUTLET_URL, {
      apiKey: process.env.OUTLET_API_KEY,
      device: process.env.OUTLET_DEVICE_NAME,
      powerState: desiredState,
    })
    .then((res) => {
      logger_.info(
        `Successfully set fridge to state (${desiredState}): `,
        res.data
      );
    })
    .catch((err) => {
      logger_.error(`Failed to set fridge to state (${desiredState}): `, err);
    });
};
```

Now we can invoke the `setFridgeState` function from our heuristic code to turn off the fridge and use the `powerCycleWaitTime` variable to tell it how long to wait before turning the fridge back on.

```js
const dutyLoop = function(){
  //...
  // Remediate the problem...
  state.lastPowerReset = now;
  setFridgeState("off");
  setTimeout(() => {
    logger_.info(
      `Powering fridge back on after (${state.powerCycleWaitTime}) minutes...`
    );
    setFridgeState("on");
  }, state.powerCycleWaitTime * 60000);
  //...
}
```

## Conclusion

We now have a system that detects when the metric is over a threshold for a specified period of time, triggers a remediation step to bring the metric back under the threshold, and then cools down before trying to remediate again.

Or in normal terms: if the freezer is getting too warm we turn the fridge off and wait 15 minutes and turn it back on. But we don’t try this more than once in an hour to let the compressor do its work.

The full source code can be found [here](https://github.com/ericvolp12/inkbird/blob/master/sample.js), but I hope nobody is in a situation where you have to use it.

This was a fun project that let me use tools from the SRE toolbox to solve an everyday problem and the best part is that it actually works! The compressor didn’t turn on last night around 1:30 AM and the system recognized it before the freezer got to 30F and corrected it without any weird bugs! I now no longer have to worry about food spoiling in the fridge overnight because the fridge forgot to cool things.

One more thing I might consider adding to the system is a notification to my email when a remediation attempt is made so I can see if this problem is getting worse or if there might be an issue with the code if it tries to remediate at high frequency.

Please note that all of this code for this project is really just duct taped together to fix my fridge and not representative of the code I typically write on “important” projects. The gohome code is a bit more polished so [go look at that instead](https://github.com/ericvolp12/gohome).

The total Bill of Materials for the project was around $50 including one Gosund outlet, one Raspberry Pi 3B+, and one Inkbird IBS-TH2.

That's all folks, thanks for reading.

---

### Eric V