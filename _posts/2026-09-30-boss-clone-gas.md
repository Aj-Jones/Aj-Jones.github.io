---
title:  "Using GAS to implement an MMO style boss fight"
image: /assets/vid/akkha_preview.gif
category: techblog
tags:  combat AI gameplay gas unreal 
permalink: /techblog/akkha/
description: "A breakdown of how to take a design spec or inspiration, and use GAS to create a boss encounter."
---
{% include elements/video.html id="syhYrrSRCyY" %}

# Overview
Hey there!

In the months prior to the layoffs at Netflix that impacted myself and the rest of Night School Studios, I was serving as the Tech Lead on a 20 member strike team established to transition the studio from Unity to Unreal Engine.
As folks ramped on and were diving into the engine, I ran intro meetings and regular help sessions to walk folks through various features, review their work, and help the team learn how to collaborate and prototype in the new engine. 

One feature I love using in UE projects, and one that was already well in use in our Strike Team, is the Gameplay Ability System (GAS).
I find it to be a fantastic framework not just for implementing your game's logic, but for helping translate ideas into their data, logic and components.

In this post, I'll be walking through an example of how GAS can be used to build a multi-phased boss fight inspired by the Akkha fight in the Tombs of Amascut raid from Old School Runescape.

By taking this reference, I'll walk through how I'd take a pitch or spec from a designer and translate that into a prototype that can be easily extended, configured, and polished into its final state.

<img src = "/assets/img/akkha/akkha_header.png" width="500" alt="">
<div align = "center"> Akkha and his UE Counterpart </div>
<br>

This post assumes some basic familiarity with GAS, and will not be covering all of the basic setup. 
If you are new to the Gameplay Ability System and looking to expand your knowledge of all the various features, I highly reccomend reviewing the [community documentation here.](https://github.com/tranek/GASDocumentation)

> You've been warned! I'll be focusing entirely on the Boss's gameplay logic, which means you'll be suffering through my engineer art with me!

My goal isn't to demonstrate "the one way to use GAS" or a step by step tutorial, but to show the flexibility GAS affords, and how the GAS framework can be used to translate a pitch into a prototype.

Let's dive in!


# The Pitch

The Akkha boss fight has two main phases:
* Phase 1: Main Fight: Akkha attacks every few seconds using a varying damage type, and is vulnerable to a varying damage type. Will ocassionally trigger special attacks as the attack style changes.
* Phase 2: Enrage phase: Becomes immune to all but Melee damage, spawns clones of himself, and teleports to a new clones spot after being hit 3 times. The player needs to dodge a field of moving explosives while attempting to defeat the boss.

> In this example, Akkha and the player will share the abilities for their auto attacks. I'll cover ways to make that easy to setup in GA_MeleeAttack

The auto-attacks Akkha uses are the combat triangle in Old School Runescape. Melee, Ranged, and Magic.
He will swap between them every so often, and is immune to damage of the same style he attacks with.

<img src = "/assets/img/akkha/osrs_combat_triangle.png" width="300" alt="OSRS Combat Triangle">
<div align="center"> Who knew Rock, Paper, Scissors could be so exciting?" </div>
<br>


Now that we have an understanding of the encounter flow, it's time to break that down into the appropriate pieces.

# The Breakdown
#### Abilities
Abilities are attacks and actions taken by a character. Our auto attacks, specials, and some utility functionality will all be abilities.

![Abilities](/assets/img/akkha/akkha_abilities.png)
* `GA_CloneSpecial` - Akkha spawns clones in each quadrant. While a clone is present in a quadrant, Akkha can not be damaged from within it. Players must destroy the clone to attack Akkha. Undestroyed clones will damage all characters in their respective quadrants.

* `GA_Enrage` - Akkha spawns his clones, heals himslef, becomes immune to everything except melee, and teleports after being hit 3 times. Meanwhile, the player must avoid explosive orbs while they attempt to defeat the boss.

* `GA_MeleeAttack` - Plays an animation, and applies damage to actors within a set area.

* `GA_MemorySpecial` - Shows a pattern of 4 quadrants that the player must navigate to. After the indicators flash, the arena will explode and deal damage in all incorrect quadrants.

* `GA_OrbSpecial` - Is granted to the boss's target, tracking position and velocity. Spawns an explosive orb that explodes on contact when the player moves.

* `GA_ProjectileAttack` - Plays an animation and fires a projectile to deal damage.

* `GA_StayInRange` - Keeps the boss within the Attack range driven by the `AttackRange` attribute. Gives the character a GameplayTag of `Combat.InRangeOfTarget` when in range.

#### Data
We'll need these abilties to be aware of the boss's state and handle cooldowns, damage, and interoperability with other abilities as appropriate.
![Effects](/assets/img/akkha/akkha_effects.png)

Once again, let's break these into their component categories:

* Attributes - Float based representation of data on a character. Think Health, Damage, Attack range, etc.
    * `HealthAttributeSet` - Contains Health, MaxHealth and Damage attributes. Damage is used to recieve damage, modify for resistances, and then apply the impact to health.
    * `CombatAttributeSet` - Contains AttackRange and Damage attributes.
<br>
<br>
* Gameplay Tags - String based hierarchical labels that can be used for just about everything. i.e. `Character.State.IsDead` and `Combat.AutoAttack.Cooldown` are examples of a few tags.

> Defining Gameplay Tags as Native Tags in C++ is helps any time you may reference a tag in code. It makes references easier as you have a direct reference rather than a string that needs to be typed (and mistyped) all over, and is still accesible in editor.

* Gameplay Effects - Effects will initialize attributes, drive changes to them, and grant and remove gameplay tags as appropriate. Effects can be selectively applied, remove others, and apply additional effects based on the GameplayTags present.


* Configuration - Health Thresholds, Time Between attacks, which specials are available, etc. These are all loose configuration variables that help designers tune the boss encounter.

#### Components and UE Objects
![Objects](/assets/img/akkha/akkha_objects.png)
For Akkha's quadrant based specials, we'll make a convienience class that contains the spawn points, colliders and indicators for the quadrant, as well as logic for spawning clones and handling Akkha's immunity. `BP_AkkhaQuadrant`. These will be used in the Simon Says, Clone, and Enrage Specials to show the appropriate visuals for the special attack, apply damage to characters in the quadrant, and apply immunity to Akkha when appropriate.

To access these quadrants, we'll make a boss singleton blueprint that lives in the scene. It will contain the references to the in world objects, and have the logic to spawn the boss. `BP_AkkhaManager`.

For the Projectile Auto Attack, we'll need a projectile to handle collision, damage dealing, and different visuals based on type. `BP_ProjectileBase`
These will be spawned by `GA_ProjectileAttack`.

For the Orb Attack, we'll need orbs that explode on contact and deal Damage over time to the player. `BP_ExplosionOrb`

Additionally, we'll need Character and Controller classes for Akkha to handle possession of the pawn, and grant the correct abilities. `BP_Akkha & BP_Akkha Clone & Controllers`

For the player we'll use the default character with the same auto-attacks Akkha uses, as the player is not our emphasis here.


# Initial Setup and Implementation

Now that we have a good idea of all the pieces we need, we can get to work.

I started with my `AttributeSets` and `GameplayTags`. Having these identified will help us configure our abilities and effects with some structure in mind.

HealthAttributeSet
* Health
* MaxHealth
* Damage

CombatAttributeSet
* AttackRange
* Damage

<div align="center"> Gameplay Tags </div>

![GameplayTags](/assets/img/akkha/akkha_tags.png)

The tags listed here will help us define what an ability is, when its allowed to run, and various runtime queries to handle game logic.

To see how that works, let's take a look at implementing an ability, in this case `GA_MeleeAbility`


![Melee](/assets/img/akkha/akkha_melee.png)
This is a pretty straightforward ability which when executed it plays an Animation, and waits for an AnimNotify to attempt to apply damage.

The interesting logic is within the `ApplyDamage` function.

![Melee](/assets/img/akkha/akkha_melee_dmg.png)

Here you can see we source the radius for the sphere trace from the AttackRange attribute. By doing this rather than hard coding, we can ensure any changes to the core attributes are respected in this logic.

> Notice all the references to Character come from the Avatar Actor from Actor Info. During initialization of the Ability System Component, have an established standard of what AvatarActor and OwnerActor will mean in your project. In mine the owner is the PlayerState, and the AvatarActor is the Character they are controlling.

#### Handling Damage Types and Immunity
Remember the combat triangle, and damage types? 

The `Add Combat Data to Spec` function used above takes our outgoing damage object, checks which type of damage the attacker is using (by checking its gameplay tags for those that begin with COMBAT_TYPE), and then applies them to the damage spec. This lets us easily ensure the damage can be applied/prevented when appropriate.

```
void UGASUtilFunctionLibrary::AddCombatDataToSpec(FGameplayEffectSpecHandle& SpecHandle, UAbilitySystemComponent* AbilitySystemComponent)
{
	FGameplayTagContainer CombatTypeContainer(COMBAT_TYPE);
	FGameplayTagContainer ResultContainer;
	
	AbilitySystemComponent->GetOwnedGameplayTags(ResultContainer);
	ResultContainer = ResultContainer.Filter(CombatTypeContainer);

	for (FGameplayTag Tag : ResultContainer)
	{
		SpecHandle.Data->AddDynamicAssetTag(Tag);
	}
}
```
> Tossing this into a Blueprint Function Library makes re-use across the project a breeze!

Rather than needing to determine if the player is immune before sending the damage spec, we can instead modify our `HealthAttributeSet` to take immunity into account.
As previously mentioned, Damage is a meta-attribute, meaning it applies changes to another attribute. 
> For more information on Meta Attributes, [check here!](https://github.com/tranek/GASDocumentation#concepts-a-meta)

This lets us perform modification on the damage value before eventually applying the change to the Health attribute. 

Here is a snippet of my calculation, being done in `UHealthAttributeSet::PostGameplayEffectExecute`

```
if (Data.EvaluatedData.Attribute == GetDamageAttribute())
{
    float LocalDamageDone = GetDamage();
    SetDamage(0.f);
    if(LocalDamageDone > 0.0f)
    {
        int RequiredImmunity = 0;
        int TypeTotal = 0;
        
        if (SpecAssetTags.HasTag(COMBAT_TYPE_MELEE))
        {
            TypeTotal++;
            RequiredImmunity ++;
            if (TargetTags.HasTag(COMBAT_PROTECTION_MELEE))
            {
                RequiredImmunity--;
            }
        }

        //Repeats for Range and Mage

        float DamageScalar = TypeTotal  > 0 ? RequiredImmunity/TypeTotal: 1.f;

        if (LocalDamageDone > 0)
        {
            const float OldHealth = GetHealth();
            const float NewHealth = OldHealth - (LocalDamageDone * DamageScalar);
            const float ModifiedHealth = FMath::Clamp(NewHealth, 0.f, GetMaxHealth());
            SetHealth(ModifiedHealth);
        }
    }
}
```

With this in place, any damage done that a player has immunity to is easily ignored.

Gaining immunity is as simple as granting the effect with the correct tag. Here's `GE_MeleeDefence`
![MeleeProtect](/assets/img/akkha/ge_meleeprotect.png)

> If you are sure you're applying your effects, but queries are failing, make sure your GE is set to Infinite. Instant effects DO NOT grant their tags to the target of the effect.

#### Showing Immunity Status

To expand one step further, we can add a gameplay cue to this effect so that an indicator is shown when the character has an active immunity.
All we have to do is set the cue in `GE_MeleeProtect`
![MeleeCue](/assets/img/akkha/cue_melee.png)

Then create a gameplay cue and assign the same tag in its class settings.
![MeleeCueImplementation](/assets/img/akkha/cue_melee_implementation.png)


#### Taking the concepts to other abilities


##### Projectile Ability

Using our `BP_ProjectileBase` class, we can make another attack ability that spawns a projectile instead of doing a sphere trace.
![Projectile](/assets/img/akkha/projectile_spawn.png)

We'll set the projectile's owner to be the ability owner, and assign our Attack Style (either Ranged or Mage), and assign an appropriate target.
If the ability is being used by the boss, IsPlayer is false, and we simply just get the player.
Otherwise, we trace for a target, and assign it if so. This is then set as the homing target on our projectile.

IsPlayer and the Attack Style variable are easy tag queries to run in ability initialization.
![TagData](/assets/img/akkha/tag_data.png)


Finally, for the projectile to actually apply damage, we simply check overlap to ensure we aren't hitting our owner, and that we're hitting something with an `Ability System Component`, and apply the damage spec accordingly.

![ProjectileDamage](/assets/img/akkha/projectile_damage.png)

> Don't forget to commit your ability and end it at appropriate times. For projectile, I start the animation and only commit once the notify is recieved. The ability is ended as soon as the animation finishes or is interrupted.

##### Abilities that track

#### Interaction Between Abilities

Now that we have our core pieces in place, I'll cover how to use tags to control interaction between abilities.

For our Melee ability, we want to ensure that the boss is in range of its target, and that it is in its melee style.

![MeleeRequirements](/assets/img/akkha/melee_requirements.png)

`AssetTags` define this abilities data. This can be used by the other Tag configuration shown here to block/cancel, and otherwise control abilities as desired.

`RequiredTags` on an ability enforce the owner needing to have the listed tags to activate this ability.

I've implemented a custom `CanCommitAbility` function, as I bumped into a scenario that just `RequiredTags` didn't support.
For the Boss, we want to make sure it's in range of its target, but the player doesn't have a target and should be able to melee whenever.

Here we can check if we are either the player, or are in range of our active target before being allowed to activate the ability.

`GA_StayInRange` is a persistent ability that continuously runs, unless blocked by tags. It provides the `Combat.InRangeOfTarget` tag when near the target, and moves the boss to the target otherwise.
![StayInRange](/assets/img/akkha/ga_stayinrange.png)


Here is an example of the Ability Tags for `GA_Enrage`.
![EnrageTags](/assets/img/akkha/enrage_tags.png)
Here, we cancel and block all other abilities, as Enrage takes priority over everything.

For more coordination with a wide variety of characters, and not as bespoke for a single purpose, consider creating Attack channels using `Gameplay Tags`, such as 
* Combat.PrimaryAttack
* Combat.SecondaryAttack
* Combat.SpecialAttack1
* Combat.SpecialAttack2
* Combat.SpecialAttack3

So you can block entire sets of abilities without needing to supply every possiblity.

## Hooking it all up

We've got some abilities, and attributes, but these need to make their way to the characters. Characters should already be setup with their `Ability System Components` and `Attribute Sets`, if not [here is a good community tutorial](https://dev.epicgames.com/community/learning/tutorials/8Xn9/unreal-engine-epic-for-indies-your-first-60-minutes-with-gameplay-ability-system) covering initial setup.

As mentioned previously, I'm using a singleton to manage this boss encounter, `BP_AkkhaManager`

![AkkhaManager](/assets/img/akkha/akkha_manager.png)

It contains the designer tuneable variables that control the overall encounter, references to in world objects used by abilities and ensures those objects exist. Abilities can simply find this one object to interface with the level hazards and ability tuning.

> Once prototyping is completed, moving these variables into a data asset for easy searchability and config is a big QOL boost!

The next piece in the chain is the `AI Controller` for our boss. It simply grants our abilities to the boss as desired.

![AkkhaController](/assets/img/akkha/akkha_controller.png)

I've split them into two categories:
* Granted Inactive Abilities - Given to Akkha, can be activated whenever.
* Abilities to Activate - Persistent Abilities, always running.

Along with `GA_StayInRange` I created one more Persistent ability. `GA_AkkhaBrain`

Rather than adding another chunk of tech in here, I opted to stay entirely within the GAS framework.
All these abilities can be triggered out of State Tree, Character Blueprints, or Behavior trees, but we'll manage it all through `GA_AkkhaBrain` here.

AkkhaBrain starts by setting the initial attribute data `GE_AkkhaStartingData` initializing its health and damage.
It sets up timers for when it should attempt to attack and when it should switch styles,
and finally listens to health changes.

With just those few functions, and our ability set, we can now script this boss fight with no problem.

![AkkhaBrain1](/assets/img/akkha/akkha_brain_1.png)
Attacks and Specials are all triggered via tag, and use the `Required/Blocked Tag` configuration and cooldowns to ensure abilities are being triggered in the correct scenario.

Switching attack styles is done by applying the correct `Gameplay Effect`.
![AkkhaBrain2](/assets/img/akkha/akkha_brain_2.png)

Each time we switch styles, we flip a coin and choose one of our other specials to activate.
This updates our `Combat_Type` gameplay tag, and the next time we attempt to AutoAttack, both the Projectile and Melee abilities will check our tags, and the appropriate one will activate.

`GA_OrbSpecial` is a unique case, as we apply the ability to the Target character, and clear it when the special is over.
![AkkhaOrb](/assets/img/akkha/akkha_orb.png)

Finally, when we reach enrage, we stop our timers, and activate our Enrage ability.
![AkkhaEnrage](/assets/img/akkha/akkha_enrage.png)


# Information Overload: Wrap up and review

Whew! Lots of moving parts! One common criticism of GAS is the amount of assets that get created.
While it is true that you will be making a fair amount, it's nothing sensible folder structure and data management can't take care of.

The real strength comes from many simple assets having the flexibility and power to be used in conjunction with one another.
On [The Foglands](/retros/foglands) I used GAS to manage our roguelite abilities and upgrades. The player unlocked cards that awarded effects and abilities to modify damage and trigger addtional abilities. Using `GameplayTags` to define our core Gameplay verbs (Shoot, Punch, Jump, Hit) let us easily make abilities and effects that responded to those tag events. This led to some awesome emergent gameplay and wacky combinations that regularly suprised us.

I hope you see the ease at which you can get dynamic encounters scripted up, with just a bit of planning and data identification.
If you have any questions on the abilities and effects I did not cover, please do not hesitate to reach out!

tldr;

* Start by identifying your desired actions, these will become your `Gameplay Abilities`.
* Determine what data is needed for those actions to function and for them to apply the desired impact to the game state. These will become `GameplayEffects` and `GameplayTags`.
    * Create your `Attribute Sets` to group attributes of similar domains together.
    * Implement custom processing of effects in `UAttributeSet::PostGameplayEffectExecute`
* Using `Gameplay Tags`, identify any relevant statefulness needed for abilities to activate, and configure your `Gameplay Ability` class defaults as appropriate.
    * Use both `Requirement tags`, Cooldown effects, and `CanActivateAbility` overrides to control when an ability is allowed to activate.
    * Use `Blocked tags` to ensure specified abilities are not allowed to trigger while the current one is executing. `Cancelled tags` to cancel ones currently active.
* Sequence the encounter using a persistent `Gameplay Ability` that triggers the desired logic, and responds to attribute and tag changes.


This project was really enjoyable to work on, and was a great test case to build out this tutorial-lite post.
If you have any further questions on GAS as a result of my information dump, please feel free to reach out on LinkedIn (linked below)

Note to Prospective Employers:
I am currently looking for my next role, and would love to discuss how I can dive in and get to work with a new team. 
I'm interested in all engineering roles across all domains, both in and out of games, as well as lead and managerial roles.

-Aj

























