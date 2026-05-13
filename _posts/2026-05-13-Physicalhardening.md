---
title: "What recent military conflicts teach us about Kinetic Resilience"
excerpt: "Traditional data center resilience assumes the building survives. Kinetic resilience assumes it does not. Kinetic resilience shifts the unit of survival from the facility to the workload, using geographic distribution, elimination of single points of failure, graceful degradation, and physical hardening to ensure services continue even when the building does not"
categories:
  - cloud
  - security
tags:
  - security
layout: single
toc: true
toc_sticky: true
mermaid: true
sidebar:
  nav: categories
---

# What recent military conflicts teach us about kinetic resilience

---

I have always been drawn to military history. The strategies, the engineering, the way wars force innovation at a pace that peacetime never does. From Alexander to the [General Bernard Montgomery](https://en.wikipedia.org/wiki/Bernard_Montgomery), I find myself reading about how leaders and armies adapted to new threats in real time. So when the conflict in Ukraine began reshaping land warfare in Europe, I followed it closely. Not the politics of it, but the engineering of it. Specifically, how aerial threats have rendered the Main Battle Tank, a platform that dominated land warfare for a century, vulnerable in ways its designers never anticipated.

The Leopard 2, the T-90, the Challenger, the M1 Abrams are some of the best of the breed Main Battle Tanks. It does not matter whose flag was painted on the hull. A first-person-view drone costing a few hundred dollars, piloted by a soldier with a headset and a gaming controller, can disable or even destroy a sixty-tonne machine worth several million. The threat comes from above. The armour was designed for the front and the sides.

What fascinated me was the response and not the destruction. Crews in the field, with limited resources and no time, began improvising defences. Welded metal cages on turret roofs. Netting draped over vehicles. Electronic jammers strapped to hulls. These were not elegant solutions. They were born of necessity, built from whatever was available, and they worked well enough to keep crews alive and protect their equipment.

Then in March 2026, military strikes damaged cloud data centre facilities in the Middle East for the first time. The threat I had been watching on the battlefield had arrived at the doorstep of digital infrastructure. In my[previous post](https://r2rajan.github.io/cloud/security/2026/05/02/KineticResilience/), I explored how to architect workloads to survive the loss of a facility. But that post deliberately left one question underexplored. Can the battlefield improvisations, be adapted to add a layer of physical protection to data centre itself?

This post is my attempt to answer that question. I lean more on curiosity and creative thinking than hard facts or core engineering. Consider this post as a thought experiment, not a technical specification.

---

## The Roof Nobody Thought About

Data centre physical security is a mature discipline. Perimeter fencing with anti-climb measures. Vehicle bollards rated to stop a lorry at speed. Mantraps with biometric authentication. Security operations centres monitoring every door and corridor. Access control systems that would make a bank vault envious. All of these measures were focused on the ground.

The roof, by contrast, is where the HVAC systems sit. Where the skylights are. Where the cable trays run. It is protected against weather, against water ingress, against the occasional bird strike. It is not protected against a deliberate aerial threat.

This was perfectly reasonable for decades. The threat model for a data centre did not include someone flying an explosive device into the roof at 120 kilometres per hour. That threat model has now changed. The Ukraine conflict demonstrated that small, inexpensive drones can deliver shaped charges with precision. The recent conflict in middle-east on cloud infrastructure confirmed that data centres are real targets.

---

## Lessons from the Battlefield

The soldiers in Ukraine did not have the luxury of waiting for a perfect solution. They needed something that worked today, built from materials they could source this week. The data centre industry has more time and more resources, but the engineering principles are the same.

![Data center Physical Hardening](/assets/images/20260513/datacenter.png)

### Netting as a First Line of Defence

In Izyum, in northeastern Ukraine, high-tensile netting is suspended over civilian infrastructure to protect against daily FPV drone attacks. The nets serve three functions. They physically trap incoming drones, entangling rotors and arresting forward motion. They can detonate a drone's payload at a safe distance above the roof surface, dissipating the blast energy before it reaches the structure. And they create uncertainty for the drone operator, who cannot be certain whether the payload will reach the intended target.

For a data centre, the same principle applies. Netting suspended above the roofline, at sufficient height to create a detonation gap, provides a passive defence layer that requires no power, no operator, and no maintenance beyond periodic inspection. It does not stop every threat. But it raises the difficulty and reduces the probability of a clean strike.

### Slat Armour for Critical Rooftop Equipment

Tank crews in Ukraine weld rigid metal grids, known as cope cages or slat armour, to their turret roofs. The engineering is straightforward. A shaped charge warhead, the type carried by most FPV drones, requires a specific standoff distance to form its penetrating jet. A metal grid detonates the warhead prematurely, before it reaches the optimal standoff distance, causing the jet to disperse rather than penetrate.

Data centre roofs have specific vulnerable points. HVAC units, skylights, cable penetrations, and exhaust vents. These are the points where a shaped charge could breach the roof envelope and damage equipment below. Rigid metal grids installed over these points replicate the cope cage principle. They do not make the equipment invulnerable. They make a successful penetration far less likely.

### Electronic Countermeasures

The third layer is electronic. FPV drones rely on a radio link between the operator and the aircraft. Disrupt that link and the drone becomes uncontrollable. It either crashes, flies off course, or enters a failsafe mode that takes it away from the target.

Electronic warfare systems that emit interference across the frequency bands used by commercial and military drones are already deployed on vehicles in the Ukraine conflict. They create a protective bubble within which drone control signals cannot reach the aircraft.

For data centres, the challenge is specificity. A facility that jams drone control frequencies indiscriminately will also disrupt its own wireless networks, cellular connectivity, and potentially GPS-dependent systems. The implementation requires directional emission, careful frequency selection, and coordination with telecommunications regulators. It is not a simple installation. But the technology exists and is proven in the field.

### Visual Obscuration

The simplest and cheapest measure is making the target harder to find and identify. FPV drone operators navigate visually. They identify the target through a camera feed, often at speed, and guide the drone to a specific point on the structure.

Reflective netting, camouflage patterns, and visual disruption materials on rooftops interfere with this process. They do not make the building invisible. They make it harder to identify the precise aim point, which reduces the accuracy of a manual strike. Against autonomous drones that navigate by GPS coordinates rather than visual identification, this measure is less effective. Against the manually piloted FPV drones that constitute the majority of the current threat, it adds a meaningful layer of difficulty.

---

## Conclusion: The Cope Cage is Not the Strategy

If you think any of these measures, would make the datacenter completely invulnerable to an aerial threat, then you are living in a mirage. A determined adversary with sufficient resources will find a way through netting, past jammers, and around camouflage.

The soldiers in Ukraine understood this. The cope cage on a tank is not a guarantee of survival. It is a way to improve the odds. It buys time. It turns a certain kill into a probable miss. It keeps the crew alive long enough to reach cover or for the electronic countermeasures to take effect.

![Iceberg Model](/assets/images/20260513/iceberg.png)

The same logic applies to data centres. Physical hardening is not a strategy in itself. It is one layer in a defence that must also include architectural resilience at the workload level.  The netting buys you time at the roof level. The cages keep the HVAC running a bit longer. The jammers might stop the drone before it arrives at all. But none of them guarantee the building survives, which is why the workload architecture underneath matters more than any of them.

Harden the facility to improve the odds. Architect the workload to survive regardless.

---

## After Thoughts

The data centre industry has not yet had its "cope cage moment." The improvised, field driven solutions that emerged in Ukraine were born out of immediate necessity. Data centre operators have the advantage of time, resources, and engineering rigour. They can design these defences properly rather than welding them together under fire.

But the window for preparation is not unlimited. The threat has moved from theoretical to demonstrated. The engineering principles are proven. The materials and technologies exist. What remains is the decision to act.

The roof deserves the same attention we have given the perimeter. The sky is no longer empty and will no longer be considered safe based on land borders.

---
