# Backup
[itch.io](https://yrgo-game-creator.itch.io/backup)

[Git](https://github.com/CarlZeidler/YRGO22-9-Backup)
<br>
![BackupTitle](https://github.com/CarlZeidler/Portfolio/assets/113012261/7b7a4024-fe88-46de-8ca7-6ab1a6adc267)
<br>
Backup was the first non-jam group project I worked on. It's a 2D platformer where you play as a robot hacking into a server. You continously create backups (checkpoints) of yourself that you can revert to at any time in order to bypass obstacles and get around hazards.<br>
## My main contributions
### The laser
In order to simplify level creation, the laser needed to have an easily modifiable length. Initially, this meant that it would take use two different points and use a line renderer to create the laser graphic and collider inbetween them. This was later expanded upon to raycast forward until it hit a surface to then use the raycast hit when creating the line, meaning it was possible to block
off the laser using platforms and doors. <br>
<br>
//IMAGES//<br>
<br>
<details>
  <summary>The laser script</summary>

  ```

using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Laser : HackableObjects
{
    public GameObject StartPointRef;
    public GameObject EndPointRef;
    public GameObject laserSpark;
    private Animator thisAnimator;
    private EdgeCollider2D laserCollider;

    private LineRenderer lineRenderer;

    private Vector2 startPoint;
    private Vector2 endPoint;

    [SerializeField] private LayerMask ignorLayers;
    private RaycastHit2D ray;
    private Vector2 rayPointRef;

    [SerializeField] bool isActive;

    void Start()
    {
        if (!isActive)
        {
            Invoke(nameof(ToggleHackState), 0.1f);
        }
        //add to manager list
        GameManager.instance.hackableObjects.Add(this);

        lineRenderer = GetComponent<LineRenderer>();
        thisAnimator = GetComponentInChildren<Animator>();
        laserCollider = GetComponent<EdgeCollider2D>();

    }
    private void Update()
    {
        startPoint = StartPointRef.transform.position;
        endPoint = EndPointRef.transform.position;
        LaserCheck();
    }
    private void LaserCheck()
    {
        //ray from start to direction of end
        ray = Physics2D.Raycast(startPoint, endPoint - startPoint, 100, ~ignorLayers);

        //check if point moved
        if(ray && ray.point != rayPointRef)
        {
            rayPointRef = ray.point;
            DrawLaserLine();
        }
        else if(!ray)
        {
            Vector2 pos = ((endPoint - startPoint)) * 100;
            if (rayPointRef != pos)
            {
                rayPointRef = pos;
                DrawLaserLine();
            }
        }
    }
    private void DrawLaserLine()
    {
        //ray origin is in worldspace, linerenderer positions are not
        lineRenderer.SetPosition(0, StartPointRef.transform.localPosition);

        //ray point - endpoint worldposition+worldpoint refrence,magic numbers
        lineRenderer.SetPosition(1, rayPointRef-endPoint+ (Vector2)EndPointRef.transform.localPosition);

        //Places laserSpark on raypoint position
        laserSpark.transform.position = new Vector2(rayPointRef.x, rayPointRef.y);
        if (ray && isActive)
            if (ray.collider.CompareTag("Player"))
                ray.collider.GetComponent<PlayerRespawn>().Die();
            else if (ray.collider.CompareTag("Enemy"))
            {
                ray.collider.GetComponent<GuardRespawn>().Die();
                if (ray.collider.GetComponent<GuardBehaviour>().objectState != ObjectState.bluePersistent)
                    ray.collider.GetComponent<GuardRespawn>().Respawn();
            }
    }

    private void LaserStatus()
    {
        if (!isHacked)
        {
            thisAnimator.SetBool("Active", true);
        }
        else if (isHacked)
        {
            thisAnimator.SetBool("Active", false);
        }
    }

    private void OnTriggerEnter2D(Collider2D collision)
    {
        if (collision.CompareTag("Player")&&isActive)
        {
            collision.GetComponent<PlayerRespawn>().Die();
        }
    }

    public void DisableBeam()
    {
        thisAnimator.SetTrigger("Deactive");
        isActive = false;
    }

    public void Reactivated()
    {
        thisAnimator.SetTrigger("Active");
        isActive = true;
    }
}
```
</details>

### The guards
The guards' behavior needed to be easily adjustable in order to make them easy to place and use regardless of the level's design.<br>
The basic behavior was for the guards to move back and forth, be able to stop and scan their surroundings, and shoot at the player if the player was spotted (or got too close).

//IMAGES//

To facilitate this, I gave the guards several variables that could be adjust when the prefab was placed in the scene in order to differentiate how each guard would act. As the guards would patrol along their cycles it also became necessary to make them turn around when they hit an edge or a wall, as the player being able to alter the level's obstacles was a central level design feature.
To facilitate this, I implemented a raycasting that would check the guard's immediate surroundings as well as what the ground in front of them looked like.

//IMAGES//

<details>
  <summary>The guard's movement script</summary>
  
  ```

  using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Spine.Unity;

public class GuardMove : MonoBehaviour
{
    [Header("Guard patrol settings")]
    [Range(1,10)] public float moveSpeed = 5f;
    [Range(1, 10)] public float maxSpeed = 10f;
    public float patrolTime;
    public float HoldTimeLeft;
    public float HoldTimeRight;
    public bool stationary;
    public bool ignoreRightLedge;
    public bool ignoreLeftLedge;

    [Header("Vision settings")]
    public float eyesPivotUpRight;
    public float eyesPivotDownRight;
    public float eyesPivotUpLeft;
    public float eyesPivotDownLeft;
    public float stationaryPivotUp;
    public float stationaryPivotDown;
    public float stationaryPivotTime;

    [Header("Patrol start direction")]
    public bool startDirectionRight;
    
    public GuardBehaviour behaviourScript;
    public GameObject visuals;
    public GameObject eyes;

    [HideInInspector] public bool canMove = true;
    [HideInInspector] public bool facingRight;
    [HideInInspector] public bool playerSeen = false;
    public bool shutDown = false;
    private float realPatrolTime;
    private float realHoldTimeLeft;
    private float realHoldTimeRight;
    private float rayCastBuffer = 0f;
    private float acceleration = 1.5f;
    private float pivotTimeRight;
    private float pivotTimeLeft;
    private float counter;
    

    private Vector3 targetAngle;
    private Vector3 currentAngle;
    private Vector3 startAngle;

    private Rigidbody2D rb2d;

    [SerializeField] private Transform raycastFeetLeftReference;
    [SerializeField] private Transform raycastFeetRightReference;
    [SerializeField] private Transform raycastLeftSide;
    [SerializeField] private Transform raycastRightSide;

    [SerializeField] private Animator[] animators = new Animator[2];

    void Start()
    {
        Setup();
    }

    void Update()
    {
        if (!stationary)
        {
            CheckPatrolTime();
            Move();
            EdgeCheck();
            Holding();
        }
        else if (stationary)
        {
            stationaryHolding();
        }
        FlipSprite();
    }

    private void Setup()
    {
        rb2d = gameObject.GetComponent<Rigidbody2D>();
        facingRight = startDirectionRight;
        realPatrolTime = patrolTime;
        realHoldTimeLeft = HoldTimeLeft;
        realHoldTimeRight = HoldTimeRight;
        pivotTimeLeft = HoldTimeLeft;
        pivotTimeRight = HoldTimeRight;
        startAngle = eyes.transform.localRotation.eulerAngles;
    }

    private void CheckPatrolTime()
    {
        if (realPatrolTime <= 0 && facingRight && realHoldTimeRight <= 0)
        {
            TurnAround();
        }
        else if (realPatrolTime <= 0 && !facingRight && realHoldTimeLeft <= 0)
        {
            TurnAround();
        }
        else if (canMove)
        {
            realPatrolTime -= 1 * Time.deltaTime;
        }
    }
    private void TurnAround()
    {
        facingRight = !facingRight;
        realPatrolTime = patrolTime;
        realHoldTimeLeft = HoldTimeLeft;
        realHoldTimeRight = HoldTimeRight;
        canMove = true;
    }

    private void Holding()
    {
        if (facingRight && realPatrolTime <= 0 && realHoldTimeRight > 0 && !shutDown && !playerSeen)
        {
            canMove = false;
            foreach(var animator in animators)
            {
                animator.SetBool("Walking", false);
                animator.SetBool("Idle", true);
            }

            //Steps through the selected angles to pivot the vision cone when the guard is at the right patrol apex.
            if (counter <= pivotTimeRight*0.25)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = new Vector3(0f, 0f, startAngle.z+eyesPivotUpRight);

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter > pivotTimeRight*0.25 && counter <= pivotTimeRight*0.5)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = startAngle;

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter <= pivotTimeRight*0.75)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = new Vector3(0f, 0f, startAngle.z-eyesPivotDownRight);

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter < pivotTimeRight)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = startAngle;

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else
            {
                counter = 0f;
            }

            counter += 1 * Time.deltaTime;
            realHoldTimeRight -= 1 * Time.deltaTime;

        }
        else if (!facingRight && realPatrolTime <= 0 && realHoldTimeLeft > 0 && !shutDown)
        {
            canMove = false;
            foreach(var animator in animators)
            {
                animator.SetBool("Walking", false);
                animator.SetBool("Idle", true);
            }

            //Steps through the selected angles to pivot the vision cone when the guard is at the left patrol apex.
            if (counter <= pivotTimeLeft * 0.25)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = new Vector3(0f, 0f, startAngle.z-eyesPivotUpLeft);

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter > pivotTimeLeft * 0.25 && counter <= pivotTimeLeft * 0.5)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = startAngle;

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter <= pivotTimeLeft * 0.75)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = new Vector3(0f, 0f, startAngle.z+eyesPivotDownLeft);

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter < pivotTimeLeft)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = startAngle;

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else
            {
                counter = 0f;
            }

            counter += 1 * Time.deltaTime;
            realHoldTimeLeft -= 1 * Time.deltaTime;
        }
    }

    private void stationaryHolding()
    {
        if (!shutDown && !playerSeen)
        {
            foreach (var animator in animators)
            {
                animator.SetBool("Walking", false);
                animator.SetBool("Idle", true);
            }
            //Steps through the selected angles to pivot the vision cone when the guard is at the right patrol apex.
            if (counter <= stationaryPivotTime * 0.25)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = new Vector3(0f, 0f, startAngle.z + stationaryPivotUp);

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter > stationaryPivotTime * 0.25 && counter <= pivotTimeRight * 0.5)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = startAngle;

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter <= stationaryPivotTime * 0.75)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = new Vector3(0f, 0f, startAngle.z - stationaryPivotDown);

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else if (counter < stationaryPivotTime)
            {
                currentAngle = eyes.transform.localEulerAngles;
                targetAngle = startAngle;

                currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
                eyes.transform.localEulerAngles = currentAngle;
            }
            else
            {
                counter = 0f;
            }
            counter += 1 * Time.deltaTime;
        }
    }

    private void Move()
    {
        if (facingRight && canMove && !playerSeen)
        {
            foreach(var animator in animators)
            {
                animator.SetBool("Idle", false);
                animator.SetBool("Walking", true);
            }
            rb2d.velocity = new Vector2(Mathf.Clamp(moveSpeed + acceleration * Time.unscaledDeltaTime, 0, maxSpeed), rb2d.velocity.y);
        }
        else if (!facingRight && canMove && !playerSeen)
        {
            foreach (var animator in animators)
            {
                animator.SetBool("Idle", false);
                animator.SetBool("Walking", true);
            }
            rb2d.velocity = new Vector2(-Mathf.Clamp(moveSpeed + acceleration * Time.unscaledDeltaTime, 0, maxSpeed), rb2d.velocity.y);
        }

        //Resets vision cone if it didn't have time to reset during the hold period.
        currentAngle = eyes.transform.localEulerAngles;
        targetAngle = startAngle;

        currentAngle = new Vector3(0f, 0f, Mathf.LerpAngle(currentAngle.z, targetAngle.z, Time.deltaTime));
        eyes.transform.localEulerAngles = currentAngle;
    }

    private void FlipSprite()
    {
        //Flips sprite depending on facing direction.
        if (facingRight)
        {   
            visuals.transform.localScale = new Vector3(Mathf.Abs(transform.localScale.x), transform.localScale.y, transform.localScale.z);
        }
        if (!facingRight)
        {
            visuals.transform.localScale = new Vector3(-Mathf.Abs(transform.localScale.x), transform.localScale.y, transform.localScale.z);
        }
    }

    private void EdgeCheck()
    {
        bool rayCastHit = false;

        if ((!Physics2D.Raycast(raycastFeetLeftReference.position, new Vector2(0f, -1f), 2f, LayerMask.GetMask("Ground")) && !facingRight && !ignoreLeftLedge) ||
                (!Physics2D.Raycast(raycastFeetRightReference.position, new Vector2(0f, -1f), 2f, LayerMask.GetMask("Ground")) && facingRight && !ignoreRightLedge) ||
                    (Physics2D.Raycast(raycastRightSide.position, new Vector2(0.5f, 0f), 1f, LayerMask.GetMask("Ground", "Obstacle")) && facingRight) ||
                        (Physics2D.Raycast(raycastLeftSide.position, new Vector2(-0.5f, 0f), 1f, LayerMask.GetMask("Ground", "Obstacle")) && !facingRight))
        {
            rayCastHit = true;
        }

        if (rayCastHit && rayCastBuffer <= 0)
        {
            TurnAround();
            rayCastBuffer = 0.6f;
        }
        else if (rayCastBuffer > 0)
        {
            rayCastBuffer -= 1 * Time.deltaTime;
        }
    }
}
```
</details>

<details>
  <summary>The guard's behavior script</summary>
    
  ```
    using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering.Universal;

public class GuardBehaviour : HackableObjects
{
    [SerializeField] private float killTime = 1f;
    [SerializeField] private bool pInRange = false;
    [SerializeField] private Animator[] animators = new Animator[2];
    [SerializeField] private Color detectionColor, activeColor, inactiveColor;
    [SerializeField] LayerMask ignoreLayer;
    [SerializeField] private Light2D visionCone;
    [SerializeField] private PolygonCollider2D visionCollider;
    [SerializeField] AudioSource detect, lose, killed;

    public GuardMove moveScript;
    public GuardVision visionScript;

    private bool tooClose = false;
    private void Update()
    {
        if (pInRange)
        {
            RaycastHit2D ray = Physics2D.Raycast(transform.position, (GameManager.instance.player.transform.position - transform.position), Mathf.Infinity, ~ignoreLayer);
            if (ray.collider.gameObject.layer == GameManager.instance.player.layer && pInRange)
            {
                visionCone.color = detectionColor;
                Invoke(nameof(Death), killTime);
                Invoke(nameof(Shoot), killTime - killTime / 4);
            }
        }
        else
        {
            CancelInvoke(nameof(Death));
            CancelInvoke(nameof(Shoot));
        }

        PlayerProximityCheck();
        VisionConeColor();
    }

    private void VisionConeColor()
    {
        if (moveScript.shutDown)
            visionCone.color = inactiveColor;
        else if (moveScript.playerSeen)
            visionCone.color = detectionColor;
        else
            visionCone.color = activeColor;
    }

    private void PlayerProximityCheck()
    {
        float minDist = 2f;
        float dist = Vector3.Distance(GameManager.instance.player.transform.position, transform.position);
        
        if (dist < minDist && !moveScript.shutDown && !tooClose)
        {
            if (moveScript.facingRight)
            {
                if (GameManager.instance.player.transform.position.x < transform.position.x)
                {
                    Invoke(nameof(KillPlayer), 2f);
                    tooClose = true;
                }
                else if (GameManager.instance.player.transform.position.x > transform.position.x)
                {
                    Invoke(nameof(KillPlayer), 0f);
                    tooClose = true;
                }
            }
            else if (!moveScript.facingRight)
            {
                if (GameManager.instance.player.transform.position.x > transform.position.x)
                {
                    Invoke(nameof(KillPlayer), 2f);
                    tooClose = true;
                }
                else if (GameManager.instance.player.transform.position.x < transform.position.x)
                {
                    Invoke(nameof(KillPlayer), 0f);
                    tooClose = true;
                }
            }
        }
        else if (dist > minDist && tooClose)
        {
            CancelInvoke(nameof(KillPlayer));
            tooClose = false;
        }
        else if (moveScript.shutDown)
            CancelInvoke(nameof(KillPlayer));
    }

    public void Shutdown()
    {
        moveScript.canMove = false;
        moveScript.shutDown = true;
        moveScript.playerSeen = false;
        pInRange = false;
        foreach(var animator in animators)
        {
            animator.SetTrigger("Shutdown");
        }
    }

    public void ReActivated()
    {
        //Restore functionality when the hacking time is over
       
        foreach (var animator in animators)
        {
            animator.SetTrigger("Startup");
        }

        //The animator will trigger the ResumeBehavior function
    }

    public void ResumeBehavior()
    {
        moveScript.canMove = true;
        moveScript.shutDown = false;
    }

    public void OnPlayerEnter()
    {
        pInRange = true;
        RaycastHit2D ray = Physics2D.Raycast(transform.position, (GameManager.instance.player.transform.position - transform.position), Mathf.Infinity, ~ignoreLayer);
        if (ray.collider.gameObject.layer == GameManager.instance.player.layer && pInRange)
        {
            Invoke(nameof(Death), killTime);
            Invoke(nameof(Shoot), killTime - killTime / 8);
            moveScript.playerSeen = true;
            detect.Play();
        }
    }

    public void OnPlayerExit()
    {
        pInRange = false;
        CancelInvoke(nameof(Death));
        CancelInvoke(nameof(Shoot));
        moveScript.playerSeen = false;
        moveScript.canMove = true;
        if (!GameManager.instance.player.GetComponent<PlayerRespawn>().isDead)
        {
            detect.Stop();
            lose.Play();
        }
    }

    public void TriggerExit()
    {
        RaycastHit2D ray = Physics2D.Raycast(transform.position, (GameManager.instance.player.transform.position - transform.position), Mathf.Infinity, ~ignoreLayer);
        if (ray.collider.gameObject.layer == GameManager.instance.player.layer && pInRange)
        {
            Invoke(nameof(Death), killTime);
            Invoke(nameof(Shoot), killTime - killTime / 8);
        }
    }

    private void Death()
    {
        RaycastHit2D ray = Physics2D.Raycast(transform.position, (GameManager.instance.player.transform.position - transform.position), Mathf.Infinity, ~ignoreLayer);
        if (ray.collider.gameObject.layer == GameManager.instance.player.layer && pInRange)
        {
            GameManager.instance.player.GetComponent<PlayerRespawn>().Die();
            detect.Stop();
            Invoke(nameof(killed.Play),.5f);
        }
    }

    private void KillPlayer()
    {
        GameManager.instance.player.GetComponent<PlayerRespawn>().Die();
        detect.Stop();
        killed.Play();
    }

    private void Shoot()
    {
        foreach (var animator in animators)
        {
            animator.SetTrigger("Shoot");
        }
    }
}

  ```
</details>
<br>

## Level 6
While not necessarily related to programming, I stil want to highlight this level which I designed single-handedly.<br>
It's one of the later levels in the game, and I wanted to incorporate as many features and mechanics of the game as possible in it.<br>

![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/82afb921-c288-45b3-849c-3677a5e54981)

## Summary
While I contributed to many other parts of this project and collaborated with the other project participants, these are what I consider my chief and most meaningful contributions.
