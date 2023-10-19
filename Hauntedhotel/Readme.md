# Haunted Hotel
![HauntedhotelTitle](/Assets/HauntedhotelTitle.png)<br>
<br>
Haunted Hotel is a short hotel management simulator where you as the proprietess have to make sure your guests get a good night's sleep by keeping the haunted chests in their rooms closed. In this project I was responsible for the animation of the main character, as well as the interactions with the chests and doors.
<br>
## My main contributions
### The animation
The animation is relatively simple, calling for bools in an animator depending on what the player is doing.

![Unity_I6XkBvUfwS](https://github.com/CarlZeidler/Portfolio/assets/113012261/f81389b2-5707-4b95-ba7e-a0cb341f62ed)

<details>
  <summary>The animation segment of the player script</summary>

  ```cs
using UnityEngine;

public class Player : MonoBehaviour
{
   private Animator thisAnimator;

    private void Start()
    {
       thisAnimator = GetComponent<Animator>();
    }

    private void Update()
    {
        if (Input.GetKey(KeyCode.A) || Input.GetKey(KeyCode.D))
        {
            thisAnimator.SetBool("Walking", true);
            thisAnimator.SetBool("IdleLong", false);
            thisAnimator.SetBool("Idling", false);
        }
        else 
        {
            thisAnimator.SetBool("Idling", true);
            thisAnimator.SetBool("Walking", false);
            Invoke(nameof(IdleLong), 2f);
        }

    }

    private void IdleLong()
    {
        if (thisAnimator.GetBool("Idling"))
        {
            thisAnimator.SetBool("Idling", false);
            thisAnimator.SetBool("IdleLong", true);
        }
    }
}
```
</details>
<br>

### The chests
The haunted chests which contain the ghosts of the hotel needed to activate on a timer and be interactable with by the player. I created a simple script to facilitate this.

![Unity_vmEoyrmS6P](https://github.com/CarlZeidler/Portfolio/assets/113012261/ea6af576-ceef-4ee2-8ef9-bf7963002198)

<details>
  <summary>The chest script</summary>

  ```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Chest_Script : MonoBehaviour
{
    public bool shaking = false;
    public bool activating = false;
    public bool isActive = false;
    public bool interactable = false;
    public bool deactivated = false;

    public int thisChestNumber;

    private Animator thisAnimator;
    private Animator ghostAnimator;
    private SpriteRenderer ghostRenderer;

    private Vector3 startingPos;

    private float shakeAmount = 0.02f;
    private float shakeSpeed = 30f;

    void Start()
    {
        startingPos.x = transform.position.x;
        startingPos.y = transform.position.y;
        startingPos.z = transform.position.z;

        thisAnimator = GetComponent<Animator>();
        ghostAnimator = transform.Find("Ghost").GetComponent<Animator>();
        ghostRenderer = transform.Find("Ghost").GetComponent<SpriteRenderer>();
    }

    void Update()
    {
        if (shaking || activating || isActive)
        {
            interactable = true;
        }
        else
        {
            interactable = false;
        }

        if (shaking)
        {
            shakingAnimation();
        }
    }

    private void shakingAnimation()
    {
        gameObject.transform.position = new Vector3 (startingPos.x + Mathf.Sin(Time.time * shakeSpeed) * shakeAmount, startingPos.y, startingPos.z);
    }

    public void StartShaking()
    {
        shaking = true;
        deactivated = false;
        Invoke(nameof(startActivating), 5f);
        soundManagerScript.instance.VOIDplaySound("chest", thisChestNumber, 1);
    }

    private void startActivating()
    {
        if (shaking)
        {
            activating = true;
            shaking = false;
            Invoke(nameof(activated), 4.8f);
            ghostAnimator.SetBool("GhostActivating", true);
            thisAnimator.SetBool("activating", true);
            soundManagerScript.instance.VOIDstopSound("chest", thisChestNumber);
            soundManagerScript.instance.VOIDplaySound("chest", thisChestNumber, 0);
        }
    }

    private void activated()
    {
        if (activating)
        {
            shaking = false;
            activating = false;
            isActive = true;
            thisAnimator.SetBool("active", true);
            thisAnimator.SetBool("activating", false);
            
        }
    }

    public void Deactivate()
    {
        if (interactable)
        {
            shaking = false;
            activating = false;
            isActive = false;
            deactivated = true;
            thisAnimator.SetBool("Deactivating", true);
            thisAnimator.SetBool("active", false);
            ghostRenderer.enabled = false;
            soundManagerScript.instance.VOIDstopSound("chest", thisChestNumber);
            soundManagerScript.instance.VOIDplaySound("chest", thisChestNumber, 2);
            soundManagerScript.instance.VOIDstopSound("ghost", thisChestNumber);
        }

    }

}

```
</details>

<details>
  <summary>The door scripts</summary>

  ```cs
using UnityEngine;

public class Door_Script : MonoBehaviour
{
    public GameObject doorOpen;
    public GameObject doorClosed;

    public void OnTriggerEnter2D(Collider2D other)
    {
        if (other.tag == "Player")
        {
            doorOpen.gameObject.SetActive(true);
            soundManagerScript.instance.VOIDstopSoundPlayerWalk();
            soundManagerScript.instance.VOIDplayOpenDoorSound();
        }
    }

    public void OnTriggerExit2D(Collider2D other)
    {
        doorOpen.gameObject.SetActive(false);
        soundManagerScript.instance.VOIDstopSoundPlayerWalk();
        soundManagerScript.instance.VOIDshutDoorSound();
    }
}

```
</details>
<br>

## Summary
In retrospect, these scripts are clumsy and I would make them completely different today. But it's fun to look back and see where you started.
