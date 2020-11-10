# gd_winch

## Configuration parameters
* **brakeForce** -        force winch applies to resist rotation (like a handbrake) when in IDLE state (to avoid birdsnest when walking out), pulling the line out will cause the battery to be re-charged. Setting this too high will make it very hard work walking out.
* **armedWinchForce** -   a light force applied by the winch when it is in an armed state. This force should meet the following specifications:
  1. sufficient to take up the slack (not including the stretch) in the line prior to launch 
  2. sufficient to rewind the line after pilot release (drag chute will be required on the line)
  3. light enough not to hazard the pilot if they do not release and want to land still attached (intentionally or due to aborted launch)
  4. light enough not to cause further danger to a pilot experiencing lockout.
* **launchWinchForce** -  the full launch force the winch will build up to when the pilot/operator triggers the launch.
* **rampTime** -          the time in seconds the winch will take to build tension when the launch is triggered.
* **lineStopLength** -    the length in meters when the winch should stop rewinding and drop the remaining line to the ground, this is to avoid overwinding

Note that during launch releasing the launch trigger will drop the force to **armedWinchForce** and holding it again will ramp back up to **launchWinchForce**.

## States
The winch will transition through the following states:
      
* **IDLE** -       Initial state, winch applies **brakeForce** to the drum but ignores any requests to launch. Will accept requests to rewind line slowly onto drum for initial loading of drum or when packing up. This is the only state in which the winch can be configured.
* **ARMED** -      Winch applies **armedWinchForce** to the line ready for launch, machine is now live and will accept requests to build to launch force. Winch can no longer be configured once armed.
* **LAUNCHING** -  Winch is ramping up to or already at **launchWinchForce**. Once pilot/operator exits launch sequence the state will return to **ARMED**.

After line release the winch will be in the **ARMED** state since the pilot/operator has also released the launch trigger on their controller. The winch will now be pulling the line in with **armedWinchForce** until only **lineStopLength** meters of line remain at which point the winch will remove all force, the line will drop to the ground and the winch will engage **brakeForce** as it automatically returns to the **IDLE** state.

## Remote controls
There are two remote controls, 
1. an android phone app that an operator can use when stood next to the winch, short (10 meters) range bluetooth, this is also used to configure the winch parameters above.
2. a long range radio remote control running on 869MHz strapped to the pilots hand

At least one controller needs to be used to operate the winch but it is also ok to use both simultaneously (e.g. pilot in charge with operator override)

### Data reported to remotes (both phone and hand control)
* Current force -          the force being applied by the winch right now
* Launch force -           the configured force the winch will apply during the launch sequence so you can check you have it set correctly
* Current line speed -     speed at which line is being rewound in meters/s
* Current line length -    line length in meters payed out off of the drum, i.e. zero when the winch is fully rewound.
* RSSI -                   signal strength indicators of both the phone and hand controller from the perspective of the winch so you can tell if it is safe to launch (probably not if the winch can only just hear you)
* Electrical state -       Main battery remaining capacity, voltage, battery current, motor current, etc. Hand controller also displays its own battery voltage so you don't launch with a nearly flat battery.


## Safety cutouts
The system runs a deadman loop, this means that if the winch does not hear from either controller for set timeout (say 1 second) then it returns to the **ARMED** state if it was previously **LAUNCHING**. 

The winch is managed by a small microcontroller (a tiny computer we'll call the winch manager) that manages all of the above. This communicates demand for motor torque to the motor controller which is a commercial unit. Should the motor controller not hear from
the winch manager for a set timeout (say 1 second) then it will release the motor (all power removed), this in effect extends the deadman loop all the way from the motor through all intermediate systems, wiring and radio links to the remotes, should the loop be severed at any point 
then the machine is made safe. 

The motor controller also has hard limits configured for protecting the battery and motor which it is continously monitoring. Should the motor controller itself fail the motor will in all likelyhood be released, or in the worst (extremely unlikely) 
case the battery will shorted through the motor windings. In no case will the motor spin out of control as the commutator (the motor controller) has failed and this is a brushless 3 phase motor not a brushed DC motor. When this occurs an extremely 
high current will flow instantaneously, it may well melt the motor windings but it will definitely blow a high voltage (low arc) 200AMP fuse that is built into the battery assembly. The battery is capable of pushing well over 1000Amps so would be undamaged
by such an event.
