# At the Hedge of Fury
**Itch.io**: https://emilxf.itch.io/at-the-hedge-of-fury<br>

![AthehedgeoffuryTitle](/Assets/AthehedgeoffuryTitle.png)

At the Hedge of Fury is a Global Gamejam game where you and your neighbour fight each other by throwing weeds into the other's back garden in order to keep your own garden pretty. My contribution to this project was mainly how, where and how often the weeds spawned and multiplied if they were left alone.<br>
## My main contributions
### Spawning weeds
There wouldn't be much point to weeding your garden if there weren't any weeds to weed out. I created a script to spawn weeds and one to multiply them if they haven't been uprooted fast enough.<br>

![Unity_UUTunPFwKz](https://github.com/CarlZeidler/Portfolio/assets/113012261/ea41894c-f63f-4316-a0cf-9b1c5dc7e4b9)

<details>
  <summary>The garden plot script which spawn the initial weeds</summary>

  ```
using UnityEngine;
using Random = UnityEngine.Random;


public class GardenPlot : MonoBehaviour
{
    [SerializeField] private GameObject weedObj;
    [SerializeField] private GameObject thrownWeed;
    [SerializeField] private float adjustedSpawnInterval = 5f;

    public bool harderByTime;
    
    private float weedSpawnInterval;
    private float countdownCounter;
    private int growState;
    private float growTimer;

    private void Start()
    {
        weedSpawnInterval = adjustedSpawnInterval;
        countdownCounter = 5f;
    }

    private void Update()
    {
        if (growState < 3)
        {
            CountDownToSpawn();
            IncreaseSpawnInterval();
        }
    }

    private void IncreaseSpawnInterval()
    {
        //Increases spawns successively as the match progresses
        if (countdownCounter <= 0)
        {
            adjustedSpawnInterval--;
            countdownCounter = 5f;
            if (adjustedSpawnInterval < 1)
                adjustedSpawnInterval = 1;
        }
        else
            countdownCounter -= 1 * Time.deltaTime;
    }

    private void CountDownToSpawn()
    {
        if (weedSpawnInterval <= 0)
        {
            SpawnWeed();
            weedSpawnInterval = adjustedSpawnInterval;
            Debug.Log("Spawning weed");
        }
        else
            weedSpawnInterval -= 1 * Time.deltaTime;
    }

    private void SpawnWeed()
    {
        Bounds bounds = GetComponent<PolygonCollider2D>().bounds;
        Instantiate(weedObj, RandomizeSpawnPosition(bounds), gameObject.transform.rotation);
    }

    private Vector2 RandomizeSpawnPosition(Bounds bounds)
    {
        return new Vector2(Random.Range(bounds.min.x, bounds.max.x), Random.Range(bounds.min.y, bounds.max.y));
    }

    public void HitByWeed(Vector2 hitLocation)
    {
        Instantiate(thrownWeed, hitLocation, transform.rotation);
        if (harderByTime)
        {
            
        }
    }
}

```
</details>

<details>
  <summary>The weed script that each new weed gets that make it multiply if not removed</summary>

  ```
using System;
using System.Collections;
using System.Collections.Generic;
using System.Drawing;
using Unity.VisualScripting;
using UnityEngine;
using Random = UnityEngine.Random;

public class GrowingWeedScript : MonoBehaviour
{
    [SerializeField] private GameObject weedObj;
    [SerializeField] private float countDownToNewWeed;
    [SerializeField] private Sprite[] growStateSprites;
    [SerializeField] private SpriteRenderer spriteRenderer;

    private int newWeedCounter = 0;
    private bool isGrabbed = false;
    private int growState;
    private float growTimer;
    public LayerMask invalidSurfaces;

    private void Start()
    {
        countDownToNewWeed = 5f;
    }

    private void Update()
    {
        if (!isGrabbed && growState > 2)
            CountDown();
        else if (!isGrabbed && growState <= 2)
            Grow();
    }

    private void Grow()
    {
        if (growTimer >= 5f && growState <= 1)
        {
            Debug.Log("Weed grown+1");
            transform.localScale = new Vector3(transform.localScale.x + 0.25f, transform.localScale.y + 0.25f,
                transform.localScale.z);
            growState++;
            spriteRenderer.sprite = growStateSprites[growState];
            growTimer = 0f;
        }
        else
            growTimer += 1 * Time.deltaTime;

    }

    private void CountDown()
    {
        if (countDownToNewWeed <= 0 && newWeedCounter < 2)
        {
            SpawnNewWeed();
            countDownToNewWeed = 5f;
            newWeedCounter++;
        }
        else
        {
            countDownToNewWeed -= 1 * Time.deltaTime;
        }
    }

    private void SpawnNewWeed()
    {
        Instantiate(weedObj, ChooseSpawnLocation(), transform.rotation);
    }

    private Vector2 ChooseSpawnLocation()
    {
        Vector2 direction = RandomizeDirection();
        int distance = RandomizeDistance();
        
        do
        {
            RandomizeDirection();
            RandomizeDistance();
        } while (Physics2D.Raycast(transform.position, direction, distance, invalidSurfaces));
        
        return Physics2D.Raycast(transform.position, direction, distance).point;
    }

    private Vector2 RandomizeDirection()
    {
        return new Vector2(Random.Range(-1f, 1f), Random.Range(-1f, 1f));
    }

    private int RandomizeDistance()
    {
        return Random.Range(3, 7);
    }
}

```
</details>

## Summary
This was a fun GameJam and an idea that turned out to be pretty fun to play in the end. 
