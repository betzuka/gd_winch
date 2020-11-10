# Paraglider winch

## Introduction
This document describes the system behaviour of a battery powered electric static winch for launching paragliders, either with the assistance of an operator or with the pilot also acting as operator.

At a high level the winch would be used as follows:
1. Operator switches on the winch.
2. Operator configures forces and other parameters for the type of launch.
3. Pilot walks out with winch line to launch point.
4. Pilot/Operator arms the winch and the winch takes up the slack.
5. Pilot/Operator triggers the launch sequence and winch builds force and pulls them into the air.
6. Pilot/Operator ends the launch sequence and releases the line if required.
7. Winch rewinds remaining line and returns to initial state.

The winch is a constant tension device, this means that it will apply a configured force to the line and maintain that force irrespective of ground or air speed. An extreme (dangerous) example of this would be launching in high wind results in a launch with negative ground speed. The winch only maintains the line tension it does not control the line speed which is dependent on glider, AUW, wind and thermic conditions.

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
* **ARMED** -      Winch applies **armedWinchForce** to the line ready for launch, machine is now live and will accept requests to build to launch force. Winch can no longer be configured once armed. Winch can be returned to **IDLE** state if it is decided not to launch.
* **LAUNCHING** -  Winch is ramping up to or already at **launchWinchForce**. Once pilot/operator exits launch sequence the state will return to **ARMED**.

After line release the winch will be in the **ARMED** state since the pilot/operator has also released the launch trigger on their controller. The winch will now be pulling the line in with **armedWinchForce** until only **lineStopLength** meters of line remain at which point the winch will remove all force, the line will drop to the ground and the winch will engage **brakeForce** as it automatically returns to the **IDLE** state.

## Remote controls
There are two remote controls, 
1. An android phone app that an operator can use when stood next to the winch operating on short (10 meters) range bluetooth, this is also used to configure the winch parameters above. 
2. A long range radio remote control running on 869MHz Lora modulation strapped to the pilots hand.

At least one controller needs to be used to operate the winch but it is also ok to use both simultaneously (e.g. pilot in charge with operator override). Only one instance of each type of remote will be active at any one time, e.g. two operators cannot both use their phones to control the winch simultaneously.

### Data reported to remotes (both phone and hand control)
* Current force -          the force being applied by the winch right now
* Launch force -           the configured force the winch will apply during the launch sequence so you can check you have it set correctly
* Current line speed -     speed at which line is being rewound in meters/s
* Current line length -    line length in meters payed out off of the drum, i.e. zero when the winch is fully rewound.
* RSSI -                   signal strength indicators of both the phone and hand controller from the perspective of the winch so you can tell if it is safe to launch (probably not if the winch can only just hear you)
* Electrical state -       Main battery remaining capacity, voltage, battery current, motor current, etc. Hand controller also displays its own battery voltage so you don't launch with a nearly flat battery.


## Safety cutouts
The system runs a deadman loop, this means that if the winch does not hear from either controller for set timeout (say 1 second) then it returns to the **ARMED** state if it was previously **LAUNCHING**. 

The winch is managed by a small microcontroller (a tiny computer we'll call the winch manager) that manages all of the above and communication with the remotes. This communicates demand for motor torque to the motor controller (a commercial unit) over a hard wired connection. Should the motor controller not hear from
the winch manager for a set timeout (say 1 second) then it will release the motor (all power removed), this in effect extends the deadman loop all the way from the motor through all intermediate systems, wiring and radio links to the remotes, should the loop be severed at any point 
then the machine is made safe. 

The motor controller also has hard limits configured for protecting the battery and motor which it is continously monitoring. Should the motor controller itself fail the motor will in all likelyhood be released, or in the worst (extremely unlikely) 
case the battery will be shorted through the motor windings. In no case should the motor spin out of control as the commutator (the motor controller) has failed as this is a brushless 3 phase motor not a brushed DC motor (it is not possible for the brushless motor to be driven without a working controller). When this occurs an extremely 
high current will flow instantaneously, it may well melt the motor windings but it will definitely blow a high voltage (low arc) 200AMP fuse that is built into the battery assembly. The battery is capable of pushing well over 1000Amps so would be undamaged
by such an event.

## Lockouts
It is expected that lockouts could be less dangerous with this winch as it employs instantaneous tension control. When a glider starts to get sideways its drag will increase. On a normal winch some time is required to react to this additional drag which causes the tension in the line to increase further increasing the danger of the lockout. On a normal winch the procedure may require winch or vehicle to be stopped and the line to be cut which takes even more time. 

On this winch, particularly when used in pilot/operator mode it would be expected at the first sign of lockout the pilot would release the launch trigger, the tension will be immediately reduced to **armedWinchForce** which should allow the pilot to recover. Importantly the winch tension is not dropped to zero nor the winch allowed to freewheel which would lead to birdsnests and the secondary danger of the line tangled around the winch itself.

## Unknown risks
The firmware running in the commercial motor controller could contain defects. The firmware is taken from the main stable release branch of the open source VESC project that is used worldwide by thousands of electric skateboard, efoil board, electric bikes and cars and some industrial applications.

## Line stretch and drum angular momentum (thoughts)
Dyneema, the line material, whether pre-stretched or not will behave like a spring over the 100's of meters we will be using for winching. This leads to an interesting control problem. Imagine the winch has 1000m of line loaded at 80kg, and the stretch at 80kg is 1% or 1cm per meter or 10m for our 1000m length. If the pilot/operator requests the force be reduced to 5kg instantaneously this 10m of stretch needs to be accounted for as the line with this stretch is acting like a spring pulling with 80kg against the winch pulling at 5kg in one direction and the glider drag pulling in the other. Either the glider needs to move 10m closer to the winch or the winch needs to pay out 10m of line, or some combination of the two. If the line speed is close to zero then all 10m will have to be pulled out of the winch. This will accelerate the drum in the reverse direction. Once the 10m stretch has been released the drum will still be spinning in the reverse direction since it has mass and therefore angular momentum that must be shed. The 5kg winch force will eventually decelerate and stop the drum but in the meantime it has made some number of rotations that will have payed line out into the floor (this is why the line runs onto the bottom of the drum), however this could lead to an oscillation, or worse a birdsnest.

A solution to this would lie in properly modelling the dynamic system, treating the line as a spring with a known spring constant (which can be measured in the workshop). The length of the line to the glider is known, and the line speed is known just before the force was reduced. The mass of the drum can be calculated from the amount of line remaining on it plus its empty mass. From this it should be possible to release the tension in a fast but controlled manor so there is zero angular momentum in the drum when the force reaches 5kg.

A more empirical solution would simply be to limit the rate at which tension is dropped based on the amount of stretch estimated to be in the line given its current length and force, the opposite of the force ramp up on launch, then tune this maximum deceleration rate so it cannot happen. Maybe drop the tension very quickly to begin with, say from 80kg to 20kg over 1/2 second then more gradually from 20kg to 5kg over another 2 seconds. We only need to stop the drum accelerating in the reverse direction - if we calculate from the line speed that the stretch will be repaid by the glider moving closer to the winch (which is the case under normal conditions but not in lockout) then no special action is required. 

This also begs the question, is it possible for the system to detect lockout and take action, for instance could a significant reduction in line speed without a reduction in force indicate that a lockout is about to occur? Could we build up a set of detectable dangerous situations and have the winch automatically take evasive action to make winching generally safer?

