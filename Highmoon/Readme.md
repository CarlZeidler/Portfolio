# HIGH MOON
**Itch.io**: https://yrgo-game-creator.itch.io/high-moon<br>
**Git:** https://github.com/eengdahl/VR<br>

![HighmoonTitle](/Assets/HighmoonTitle.png)

HIGH MOON is a shooting range style VR game that takes place in a graveyard where you shoot cardboard cutouts of monsters with your trusty revolver, Rolf. One of the key design ideas of the game was the focus on interactivity with the gun. Thus, you are supposed to hold the controller 
upside-down and in the wrong hand in order to use the trigger button as the hammer of the revolver. My contributions to this project where mainly the game loop and game rounds systems.<br>
## My contributions
### The game loop and round system
The game is designed so that the player will start a round, fire at targets, and then receive a score total depending on how well they did. To facilitate this, the game loop needed to have a rounds system implemented that the player could trigger and restart should they want to.

![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/9cf15595-bddc-41d7-a734-862958f1b88c)

<details>
  <summary>The game loop implementation in the game controller</summary>

```cs
using UnityEngine;

public enum GameState { inMenu, Countdown, inGame, preGame }

public class GameController : MonoBehaviour
{
    [Header("State")]
    public GameState currentGameState;

    [Header("Components")]
    public UIController uiController;
    public ScoreController scoreController;
    public TargetPlacer targetPlacer;
    public Shoot shoot;
    AchievementHandler achievementHandler;
    private AudioSource audSource;
    [SerializeField] GameObject countdownSigns;

    [Header("Settings")]
    public Difficulty chosenDifficulty;
    public bool timeTrialEnabled;
    public float gameTime = 90f;
    public float gameTimer;

    private void Awake()
    {
       currentGameState = GameState.preGame;

        uiController = FindObjectOfType<UIController>();
        uiController.gameController = this;
        scoreController = FindObjectOfType<ScoreController>();
        scoreController.gameController = this;
        targetPlacer = FindObjectOfType<TargetPlacer>();

        uiController.gameController = this;
        scoreController.gameController = this;
    }

    private void Update()
    {
        if (currentGameState == GameState.inGame && timeTrialEnabled)
        {
            gameTimer -= 1 * Time.deltaTime;
            scoreController.UpdateTimer(gameTimer);

            if (gameTimer <= 15 && !spawnedBoss)
            {
                moon.material = redMoon;
                RenderSettings.skybox = redSky;

                targetPlacer.InitializeTargets(targetPlacer.SpawnBoss(chosenDifficulty));
                spawnedBoss = true;
            }

            if (gameTimer <= 0)
            {
                moon.material = normMoon;
                RenderSettings.skybox = normSky;

                EndGame();
            }
        }
    }

    public void SetupGame(Difficulty difficulty) //spawn targets and ready the countdown
    {
        chosenDifficulty = difficulty; //set difficulty
        gameTimer = gameTime;
        scoreController.ResetScore(); //reset the score
        //scoreController.ClearAchievementsDisplay();
        targetPlacer.PlaceTargets(chosenDifficulty);
        countdownSigns.GetComponent<Animator>().SetTrigger("readyCountdown");
    }

    public void StartCountdown() //activate the countdown animation
    {
        currentGameState = GameState.Countdown;
        countdownSigns.GetComponent<Animator>().SetTrigger("startCountdown");
        scoreController.leaderboardAnim.SetBool("showLeaderboard", false);
    }

    public void ResetCountdown()
    {
        countdownSigns.GetComponent<Animator>().SetTrigger("resetCountdown");
    }

    public void StartGame() //start game
    {
        currentGameState = GameState.inGame;
        shoot.currentGameState = currentGameState;
        targetPlacer.InitializeTargets(targetPlacer.activeTargets);
        FindObjectOfType<CylinderPopulate>().FillBarrel(6);
    }

    public void EndGame() //only called when time is up, otherwise call ReturnToMenu
    {
        audSource.Play();
        targetPlacer.RemoveTargets();
        achievementHandler.SendAchievements();
        spawnedBoss = false;

        //Code to end the round and save score
        if (!scoreController.CheckLeaderboard(chosenDifficulty)) //if we're not on the leaderboard, automatically return to menu, otherwise we return throught SubmitScore
            uiController.ReturnToMenu();

        currentGameState = GameState.inMenu;
        //this needs to happen last
        shoot.currentGameState = currentGameState;

        scoreController.leaderboardAnim.SetBool("showLeaderboard", true); //show leaderboards again
    }

    public void ReturnToMenu() //only called when pressing "Choose New Difficulty", otherwise call EndGame
    {
        moon.material = normMoon;
        RenderSettings.skybox = normSky;

        spawnedBoss = false;
        currentGameState = GameState.inMenu;
        shoot.currentGameState = currentGameState;
        targetPlacer.RemoveTargets();
        scoreController.leaderboardAnim.SetBool("showLeaderboard", true); //show leaderboards again
    }

    public void BulletFired(bool wasOnTarget)
    {
        scoreController.BulletWasFired(wasOnTarget);
    }

    public void OverrideTime(int time)
    {
        gameTimer += time;
    }
}
```
</details>

### Target trails
Since, in a VR game, the game world is all around the player, targets will sometimes be out of sight for the player. In order to notify the player of targets that are "out of sight" I created a trail along the ground towards targets that had not been hit for a while.

![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/fca53a02-a9ec-4a8c-8de5-e7cfb2459993)

<details>
  <summary>The trail renderer, which is called from the targets themselves when they haven't been hit for a while</summary>

  ```cs
using System.Collections.Generic;
using UnityEngine;

public class TargetTrailsRenderer : MonoBehaviour
{
    [SerializeField] private GameObject vampireTrail;
    [SerializeField] private float amplitude = 1.0f;
    [SerializeField] private int waveCount = 5;
    private List<GameObject> activeTargets = new();
    private LineRenderer lineRenderer;
    public List<GameObject> activeLines = new();

    public void ShowLine(GameObject target, ShootableTarget.MonsterType type)
    {
        //Shows lines to targets that have not been hit for a while.
        //Called by ShootableTarget

        GameObject trail;
        waveCount += Random.Range(-1, 1);
        amplitude += Random.Range(-.015f, .015f);

        Vector3 startPoisiton = Vector3.Lerp(transform.position, target.transform.position, .15f);
        Vector3 endPosition = Vector3.Lerp(transform.position, target.transform.position, .8f);

        //Comment the below line to make trails "fly" to target instead of snaking along the ground
        endPosition = new Vector3(endPosition.x, .4f, endPosition.z);
        GameObject newLine = Instantiate(trail, startPoisiton, Quaternion.identity);
        LineRenderer lineRenderer = newLine.GetComponent<LineRenderer>();
        int maxIndex = lineRenderer.positionCount;

        Vector3[] points = new Vector3[maxIndex];

        for (int i = 0; i < maxIndex; i++)
        {
            float t = i / (float)(maxIndex - 1);
            Vector3 linePoint = Vector3.Lerp(startPoisiton, endPosition, t);

            float offset = 0f;

            for (int waveIndex = 0; waveIndex < waveCount; waveIndex++)
            {
                float waveOffset = Mathf.Sin(t * Mathf.PI * waveCount * waveIndex * Mathf.PI) * amplitude;

                offset += waveOffset;
            }

            if (i % 2 != 0)
                linePoint += transform.right * offset;
            else
                linePoint += -transform.right * offset;
            //Comment out the below line to make lines "fly"
            linePoint.y = 0.4f;
            points[i] = linePoint;
        }

        lineRenderer.SetPositions(points);
        activeLines.Add(newLine);
        target.GetComponent<ShootableTarget>().line = newLine;
    }

    private void SetupLines()
    {
        //Creates a reference to the trail renderer, done by the trail renderer itself at the start
        //of the round.
        foreach (GameObject target in activeTargets)
        {
            target.GetComponent<ShootableTarget>().trailRenderer = this;
        }
    }

    public void PopulateList(List<GameObject> newList)
    {
        //Populates list with the currently active targets
        //Called from TargetPlacer
        activeTargets = newList;
        SetupLines();
    }

    public void DePopulateList()
    {
        //Depopulates list and removes the lines when targets are removed
        foreach (GameObject line in activeLines)
        {
            Destroy(line);
        }

        foreach (GameObject target in activeTargets)
        {
            target.GetComponent<ShootableTarget>().line = null;
        }

        activeTargets.Clear();
        activeLines.Clear();
    }

    public void RemoveLine(GameObject lineToKill)
    {
        int i = activeLines.IndexOf(lineToKill);
        Destroy(activeLines[i]);
        if (activeLines[i] != null)
            activeLines.Remove(lineToKill);
    }
}
```
</details>

### Shootable UI
Rather than using the VR controller's own pointer to interact with the game's UI, we wanted to use the revolver to do it by shooting at the UI. To do this, I created a Shootable Button script that invoked the UI's normal functionality as a function when the UI is shot.

<details>
  <summary>The shootable button script</summary>

  ```cs
using UnityEngine;
using Button = UnityEngine.UI.Button;
using Toggle = UnityEngine.UI.Toggle;

public class ShootableButton : MonoBehaviour
{
    public enum UItype { buttonType, toggleType }

    [SerializeField] private UItype uiType;

    public void TriggerButton() //This function is called by the shoot script on the gun when it raycasts to detect what it fired on
    {
        switch (uiType)
        {
            case UItype.buttonType:
                GetComponent<Button>().onClick.Invoke();
                break;
            case UItype.toggleType:
                GetComponent<Toggle>().isOn = !GetComponent<Toggle>().isOn;
                break;
        }
    }
}
```
</details>

## Summary
Working in VR is very different from "normal" games. Especially when developing towards the Oculus Quest which has some strict hardware limitations. Ultimately I'm glad with the result my group produced.
