# Box2D for platformer physics

*Benefits and challenges of using Box2D for Mystic Melee's physics.*

I decided to use the somewhat ubiquitous Box2D physics engine for Mystic Melee's physics. As common as Box2D is, I've seen advice against using it for platformer games. In fact, the first thing Google comes up with for "box2d platformer" is Why Using A Physics Engine For A 2D Platformer Is A Terrible Idea. It definitely takes some wrangling -- the rigid-body physics simulation of Box2D doesn't lend itself to classically (and unrealistically) tight platformer controls. However, if you want your game to have robust and realistic physics interactions beyond your platforming character, getting Box2D to work for your player character is worth it.

My goal was to enable level designs with surfaces at any angle, including surfaces that rotate or move. The player should transition seamlessly from walking on flatter surfaces to sliding on inclines. The characters should be able to crouch and slide with a smaller hitbox. The movement for the character should feel tight and responsive, and not like you're just sliding a box around. I'll go through how Mystic Melee solves these platformer problems with Box2D.

The excellent iforce2d tutorials are recommended as a starting point for working with Box2D. If you don't know Box2D's terminology of bodies, fixtures, sensors, etc., learn about them there!

The player character Body

```
b2Vec2 velocity = m_unit->m_physics->Body().GetLinearVelocity();
if (m_movingPlatform)
{
   velocity -= m_movingPlatform->GetLinearVelocity();
   if (m_movingPlatform->GetAngularVelocity() != 0)
   {
      b2Vec2 contactVector = m_movingContactPoint - m_movingPlatform->GetPosition();
      b2Vec2 contactAngularVelocity = b2Vec2(-1 * m_movingPlatform->GetAngularVelocity() * contactVector.y, m_movingPlatform->GetAngularVelocity() * contactVector.x);
      velocity -= contactAngularVelocity;
   }
}
```

The Box2D Body for the player character is made of the 5 fixtures above.

Hitbox

The hitbox fixture is a rectangle that's slightly smaller than the player sprite. This fixture is used to physically interact with the world by colliding with the ground, solid objects, or spells. Usually, this fixture is set to have fixed rotation so the player stays upright, although this is turned off if you've been frozen solid or reduced to a decapitated head.

This fixture is replaced by ones of different sizes if you're crouching or sliding, allowing you to get through tight spaces or avoid enemy attacks.


(weird colors courtesy of licecap)

The most important property of the hitbox fixture is the friction, and it's modified constantly. If the player is not trying to move left or right, the friction with the ground is set very high so the player stops moving fairly quickly. When trying to move or trying to slide, friction with the ground is very low. This allows the player to keep their momentum if sliding and allows the force generated from walking to push you along the ground. I use this change in friction alone to keep the movement responsive. Some suggest using opposing forces to slow down the character if no buttons are being pressed, but I don't see this as neccessary unless you want your character to stop while airborne.

Note that to ensure the friction is set properly between the player and a floor, it needs to be set in every PreSolve collision callback between the hitbox and a floor fixture. If you only modify the hitbox fixture's friction, the collision may use an outdated friction coefficient (e.g. if you were not walking when you landed on the floor, and then start walking without recontacting it).


Foot sensor

I should note that all floors/walls/platforms in Mystic Melee are made of b2EdgeShapes. These edges are connected using "ghost verticies" as described here. This is an important part of level building because otherwise the player's hitbox will occasionally "catch" where one edge meets another, slowing you down and popping you into the air inexplicably.

The foot sensor uses BeginContact and EndContact collision callbacks to keep a list of the floors you're standing on. When the edge shapes are loaded into the level, their UserData stores the angle of the edge. This allows the foot fixture of the player to quickly determine what you're standing on. If you're standing on multiple edges, it resolves to the flatter one. On opposing edges, it resolves you to be standing on flat ground so you can stand at the top of steep peaks or bottom of wells.



Past a certain angle, you will slide down slopes like the ones above. At that point, they are treated like walls, although the angle you jump off from is more forgiving.

Getting this system to work with moving platforms is pretty simple. The game just has to calculate the angle of the edge on the fly if it is rotating.



While determining the angles of moving/rotating platforms was simple, making the movement and animations work nicely on these platforms took a little more work. If the player is standing on a moving platform, their global velocity won't be correct for the purposes of running speed and whether they should have the running animation displayed. We'll need to get the player's velocity in reference to the point on the platform where they are standing. I use the following code to account for the platform's linear and angular velocity to get the player's final correct velocity used to determine running capabilities and animations.
```
b2Vec2 velocity = m_unit->m_physics->Body().GetLinearVelocity();
if (m_movingPlatform)
{
   velocity -= m_movingPlatform->GetLinearVelocity();
   if (m_movingPlatform->GetAngularVelocity() != 0)
   {
      b2Vec2 contactVector = m_movingContactPoint - m_movingPlatform->GetPosition();
      b2Vec2 contactAngularVelocity = b2Vec2(-1 * m_movingPlatform->GetAngularVelocity() * contactVector.y, m_movingPlatform->GetAngularVelocity() * contactVector.x);
      velocity -= contactAngularVelocity;
   }
}
```


Left, Right and Head sensors

These sensors are simpler. The wall sensors determine (through Begin and EndContact collision callbacks) if you are touching a wall. When touching a wall and no floors, the player can walljump or hold against the wall to slow their fall.

The head sensor is used when jumping to stop upward force from the jump. I've implemented jumps in the traditional Mario style -- when the player holds the jump button, the player continues to create upward force for a number of frames after leaving the ground. This makes for an easy way to control your jump height. However, if the character's head hits a ceiling, we should cut off additional jump forces, otherwise the player could stick to ceilings for a few frames.

---

I think that'll wrap up part 1 of this post on Mystic Melee's platforming physics in Box2D. I'll leave the specifics of movement and some extra tricks for smooth movement for part 2. Hope this is helpful to someone else interested in making platformers with Box2D!