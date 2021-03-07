---
layout: post
title: "How SREs Solve Problems at Home (Making my Dumb Fridge Smart)"
---

## Pretext

This is a story partially about laziness but not the weird kind of laziness where, instead of doing the thing, you decide to do something completely different and more convoluted that ends up taking way more time because you can't be bothered to do the thing.

In this instance, the thing is fixing my fridge (though I would argue it shouldn't be my responsibility to do that, I rent and my landlord should have fixed this when it became an issue months ago but oh well).

I am constantly reminded why I enjoy software engineering and how the tools I have in my engineering toolbox can solve a lot more problems than I usually think they can.

## My Fridge is Broken

On the list of essential appliances in a home, at the top you'll find a refrigerator.

In my apartment, constructed back in 2003, most of the appliances are holdovers from back then, meaning at this point, my fridge is old enough to vote. While that's a fun and quirky fact, somewhat less fun and less quirky is that my fridge is broken. I have somewhat diagnosed that the problem lies in the connection between the temperature control unit and the compressor. The compressor works fine, but every few days the fridge will decide it doesn't need the compressor anymore and just won't tell it to turn on.

I learned that my fridge was broken by opening the top-portion's freezer and looking at a puddle in my ice bin. While this was a useful signal to notice something was wrong with my fridge, I didn't realize what was wrong or how I could fix it. After conversation with my landlord he said "Oh yes the previous tenant had an issue and whenever that happened they cut power to the fridge and waited about 15 minutes and then turned it back on".

Well, at least now I had a remediation step to take, so I gave it a shot and, don't you know, it worked perfectly! Problem solved!

...Not really though, unless I want to be checking on my ice bin hourly to know if I need to cut the power and then cutting it, setting a timer, and restoring it on the regular, I needed another solution.

At this point a normal person might consider trying to repair the fridge or hiring someone to do as much, but it's been over a month since the fridge technician made his house call and there has been no follow-up while the issue persists, so I hopped on amazon and started to solve the problem the only way I know how: with metrics and automated remediation.

## The Metric

For this problem, there are several metrics we could consider. Initially, my metric was looking in the ice bin to see if it was melting (if it was, this would lead to it all refreezing once the compressor started up again and turn my cubes into a giant chunk of ice). While this metric works well for me, a human, it also catches the problem too late (the ice has already begun to melt meaning the freezer is over 32F) and it is harder to measure with a computer (maybe a trained computer vision model could learn to see melting ice differently from non-melting ice but that's a bit more than I signed on for).

The solution I settled upon was to monitor the temperature inside the freezer with some kind of probe, though I could have also done something like monitoring power draw at the wall for the fridge etc.

So, I picked up a cheap bluetooth thermometer [here](https://www.amazon.com/gp/product/B08S3CGZ3Q) ($15 at time of purchase though the $20 model has humidity reading too) and set to work installing the app and checking it regularly.

This let me observe temperatures that the freezer oscillates between during normal operation and what it looks like when the temperature begins to run away.

![Fridge temperature graph showing oscillations between 2.5F and 12F excepting one large spike to around 28F. Oscillations appear to occur every hour or so](/public/images/2021-03-07/fridge_temps.png)

As you can see, the temperature sits somewhere from 2.5F to 12F regularly and occasionally spikes during weird events. In the image shown, that spike was halted by the magical system I have completed but you only know a small part about for now.

So, we now have a metric viewable in an app, let's grab some random FOSS project that can talk to these weird thermometers over bluetooth and slap in in a Raspberry Pi.

## Gathering Data

To interface with the thermometer, instead of using the weird limited app, I wondered if someone else had taken a crack at talking to this device and, it turns out, they did. I found a project called Inkbird [here](https://github.com/tobievii/inkbird) and [forked it](https://github.com/ericvolp12/inkbird) to twist to my nefarious purposes of making my fridge smart.

In a nutshell, this is a Node.JS project that utilizes the [noble](https://www.npmjs.com/package/@abandonware/noble) BLE module to talk to Inkbird thermometers, though this code was written for a previous version so I had to modify one of the constants to scan BLE devices and filter for the right name (I found the right name through experimenting with a bluetooth scanning app on my phone and opening and closing the freezer door to see which device jumped in signal when it opened).

I wrote a duty loop that scans for the thermometer once a minute to avoid killing its battery prematurely, grabs one datapoint containing a temperature and battery level reading, and then disconnects. It then exposes the program state via a JSON API endpoint with Express.

An abbreviated form of the final product can be found below.

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

To enhance it, we want to add some kind of heuristic to determine if the freezer is too warm.

We can do that by setting some kind of trigger temperature, then a number of minutes the freezer must be above that temperature to trigger the alarm. We also want to add an alarm cooldown so we don't trigger once a minute while the freezer temp is rising, even if we kick in remediation, it can take some time to drop the temps back down.

So we'll add these variables to `state` (though really they're constants and should be someplace else but I haven't gone through to polish this code up yet nor will I really...).

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

Nice, now we want to ad the heuristic to our duty loop by grabbing the last `triggerMinutes` worth of measurements from `sensorState.temps` and checking if all of them are over the `triggerTemp`.

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

To remediate the issue, we know that powering the fridge off and waiting about 15 minutes and turning it back on seems to work. Do I know why that works? I have some theories about things warming up enough to trigger sensors or maybe just compressors cooling down but thankfully, to fix the symptoms, I don't need to know why it works!

All I need to do is power off the fridge, wait 15 minutes, and turn it on, but from code.

So, as a bit of background I wrote this [home automation API in Golang](https://github.com/ericvolp12/gohome) a couple months ago which talks to Phillips Hue, Belkin WeMo, and Tasmota IOT devices through different protocols and gives me some ability to control them at will and with automation. The interesting bits for this project are the `SetStateHandler` which can be found [here](https://github.com/ericvolp12/gohome/blob/master/cmd/gohome/main.go#L141).

This code allows me to set the power state of a given [Tasmota](https://tasmota.github.io/docs/) outlet connected to a MQTT broker in my apartment. I hacked up some old [Gosund ESP8266 based WiFi Smart Plugs](https://www.amazon.com/gp/product/B079MFTYMV) and installed Tasmota on them with the aid of [TuyaConvert](https://github.com/ct-Open-Source/tuya-convert) which allows you to OTA upgrade some off-the-shelf IOT devices. I'll note here that some of the more recent Gosund units I bought from Amazon have updated firmware that is harder to flash and for a good chunk of time I wasn't able to update them until the TuyaConvert project managed to crack some of their newer security features for OTA updates.

Okay, so we have an API in my home that allows me to talk to an outlet and make it turn on and off, so the last thing to do is integrate it into the code.

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

So, we now have a system that detects when the metric (freezer temperature) is over some threshold for a specified period of time, effects some remediation step to bring the metric back under the threshold, and then cools down before trying to remediate again.

Or, in normal terms: if the freezer is getting too warm, we turn the fridge off and wait 15 minutes and turn it back on, but we don't try this more than once in an hour to let the compressor do its work.

The full source code can be found [here](https://github.com/ericvolp12/inkbird/blob/master/sample.js), but I hope none of you have to use it.

This was a fun project that let me put my SRE mindset to solving an everyday problem and the best part is, it actually works! The compressor didn't turn on last night around 1:30 AM and the system recognized it before the freezer got to 30F and corrected it without any weird bugs! I now no longer have to worry about food spoiling in the fridge overnight because the fridge forgot to cool things. 

One more thing I might consider adding to the system is a notification to my email when a remediation attempt is made so I can see if this problem is getting worse or if there might be an issue with the code if it tries to remediate at high frequency.

Please note that the code in this project is really just duct taped together to fix my fridge and not representative of the code I typically write on "important" projects. The [gohome](https://github.com/ericvolp12/gohome) code is a bit more polished so go look at that instead.

The total Bill of Materials for the project was around $50 including one Gosund outlet, one Raspberry Pi 3B+, and one Inkbird IBS-TH2.

I hope you enjoyed this fun IOT project, I had a blast engineering it and am very happy with the results!

That's all folks, thanks for reading.

---

### Eric V