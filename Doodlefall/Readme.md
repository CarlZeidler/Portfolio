# Doodlefall
[Git](https://github.com/CarlZeidler/Doodlefall)<br>
<br>
![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/00bd3ace-f1cc-46a8-8da4-299a7ed4b5ae)
<br>
Doodlefall was a solo mobile game project where I explored networking and database implementation in the form of a Firebase integration. It's a short puzzle game where you roll a ball down a board by tilting your phone. It utilizes the phone's built-in gyro to change the gravity of the game and features profile creation and high score upload to a Firebase database.
## Main features
### Profile creation and sign-in
In order to utilize the Firebase functionality, the player would need to create a profile (or use a temporary one as a guest). This necessitated the creation of functionality to register, login, as well as saving the user's settings, score and preferences.


|![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/ed904812-433c-47c9-9fed-ceed15540884)|![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/7c1277ff-d81a-4324-9fbc-1cf9ac2429b5)|


<details>
  <summary>The Startup Screen Script, which handles UI functionality on the game's initial screen</summary>

  ```
using UnityEngine;
using TMPro;
using UnityEngine.UI;
using UnityEngine.SceneManagement;

public class StartupScreenScript : MonoBehaviour
{
    #region SERIALIZED FIELDS

    [SerializeField] private GameObject loginCanvas;
    [SerializeField] private GameObject nameConfirmCanvas;
    [SerializeField] private GameObject playerSetupCanvas;
    [SerializeField] private TMP_InputField usernameField;
    [SerializeField] private TMP_InputField emailField;
    [SerializeField] private TMP_InputField passwordField;
    [SerializeField] private TMP_InputField anonymousField;
    [SerializeField] private TMP_Text welcomeText;
    [SerializeField] private Button signInButton;
    [SerializeField] private Button registerButton;
    [SerializeField] private Button anonymousSignInButton;
    [SerializeField] private Slider colorSlider;
    [SerializeField] private GameObject playerBall;
    [SerializeField] private MeshRenderer playerBallMesh;
    [SerializeField] private GameObject metallicButton;
    [SerializeField] private GameObject woodenButton;
    [Header("Check Script for details")]
    [SerializeField] private Material[] ballMaterials; //MetallicMaterial1, Woodmaterial1
    
    #endregion
    
    private SaveManager _saveManager;
    private GameManager _gameManager;
    private MainMenuScript _mainMenu;
    private PlayerStats _playerStats;
    private PlayerInfo _playerInfo;

    private string _registeredUserJson;
    private Color _ballColor;
    private int _ballType = 1;
    private string _playerName;
    

    private void Start()
    {
        _saveManager = FindObjectOfType<SaveManager>();
        _gameManager = FindObjectOfType<GameManager>();
        _playerInfo = FindObjectOfType<PlayerInfo>();
        _playerStats = new PlayerStats();
        colorSlider.onValueChanged.AddListener(delegate { ChangeBallColor();  });
    }
    
    private void Update()
    {
        //Test accounts:
        if (Input.GetKeyDown(KeyCode.Alpha5))
        {
            Debug.Log("Logging in test user1");
            _saveManager.UserSignIn("testuser@testaddress.com", "testtest1", SignInCallback);
        }
        
        if (Input.GetKeyDown(KeyCode.Alpha6))
        {
            Debug.Log("Logging in test user2");
            _saveManager.UserSignIn("testuser2@testaddress.com", "testtest1", SignInCallback);
        }
        
        if (Input.GetKeyDown(KeyCode.Alpha7)) 
        {
            Debug.Log("Logging in test user3");
            _saveManager.UserSignIn("testuser3@testaddress.com", "testtest1", SignInCallback);
        }
        
        SpinBall();
    }

    private void SpinBall()
    {
        playerBall.transform.Rotate(new Vector3(0, -0.1f, 0));
    }
    
    private void ChangeBallColor()
    {
        _ballColor = Color.HSVToRGB(colorSlider.value, 0.85f, 0.85f);
        playerBallMesh.material.color = _ballColor;
    }
    
    public void ToggleBallMaterial()
    {
        if (_ballType == 1)
        {
            _ballType = 2;
            playerBallMesh.material = ballMaterials[1];
            colorSlider.value = 0;
        }
        else if (_ballType == 2)
        {
            _ballType = 1;
            playerBallMesh.material = ballMaterials[0];
            colorSlider.value = 0;
        }
    }
    
    private void UISetup()
    {
        if (_ballType == 2)
        {
            metallicButton.SetActive(false);
            woodenButton.SetActive(true);
        }
    }
    
    public void SaveDataToFirebase()
    {
        _playerStats.ballType = _ballType;
        _playerStats.ballColor = colorSlider.value;
        _saveManager.SaveToFirebase(_playerStats);
    }
    
    #region INITIAL SCREEN/SIGN-IN FUNCTIONS
    
    /////////////////// USER AUTHENTICATION/REGISTRATION/SIGN-IN FUNCTIONS ///////////////////
    
    public void StartSignIn()
    {
        SaveManager.Instance.UserSignIn(emailField.text, passwordField.text, SignInCallback);
    }

    public void StartRegistration()
    {
        SaveManager.Instance.RegisterNewUser(emailField.text, passwordField.text, RegistrationCallback);
    }

    public void StartAnonymousSignIn()
    {
        SaveManager.Instance.AnonymousRegistrationIn(anonymousField.text, RegistrationCallback);
    }

    public void StartRegisteringUserName()
    {
        SaveManager.Instance.RegisterUserName(usernameField.text, UserNameRegistrationCallback);
    }

    public void ActivateButtons()
    {
        if (!string.IsNullOrEmpty(emailField.text) && !string.IsNullOrEmpty(passwordField.text))
        {
            signInButton.interactable = true;
            registerButton.interactable = true;
        }
        if (!string.IsNullOrEmpty(anonymousField.text))
            anonymousSignInButton.interactable = true;
    }
    private void RegistrationCallback(bool anonymousRegistration, PlayerStats playerStats)
    {
        _playerStats = playerStats;
        
        if (anonymousRegistration)
        {
            loginCanvas.SetActive(false);
            playerSetupCanvas.SetActive(true);
            SetWelcomeText();
        }
        else
        {
            loginCanvas.SetActive(false);
            nameConfirmCanvas.SetActive(true);
        }
    }

    private void UserNameRegistrationCallback()
    {
        nameConfirmCanvas.SetActive(false);
        playerSetupCanvas.SetActive(true);
        
        SavePlayerInfo();
    }

    public void SignInCallback(string jsonString)
    {
        loginCanvas.SetActive(false);
        playerSetupCanvas.SetActive(true);
        
        LoadPlayerInfo(jsonString);
    }
    
    public void SavePlayerInfo()
    {
        _playerStats.ballType = _ballType;      
        _playerStats.ballColor = colorSlider.value;
    }
    
    #endregion

    #region PLAYER SETUP FUNCTIONS
    
    ////////////////////////////// PLAYER SETUP FUNCTIONS //////////////////////////////
    
    
    public void LoadPlayerInfo(string jsonString)
    {
        Debug.Log("loaded json: " + jsonString);
        _playerStats = JsonUtility.FromJson<PlayerStats>(jsonString);
        Debug.Log("json converted to object: " + _playerStats);
        _playerName = _playerInfo.playerName;
        colorSlider.value = _playerStats.ballColor;
        
        SetWelcomeText();

        if (_playerStats.ballType == 0)
        {
            _ballType = 1;
            playerBallMesh.material = ballMaterials[0];
            _playerStats.ballType = _ballType;
        }
        else if (_playerStats.ballType == 1)
        {
            _ballType = _playerStats.ballType;
            playerBallMesh.material = ballMaterials[0];
        }
        else if (_playerStats.ballType == 2)
        {
            _ballType = _playerStats.ballType;
            playerBallMesh.material = ballMaterials[1];
        }
        
        UISetup();
        
        playerBallMesh.material.color = Color.HSVToRGB(colorSlider.value, 0.85f, 0.85f);
    }
  
    private void SetWelcomeText()
    {
        welcomeText.SetText("Welcome " + _playerInfo.playerName + "!");
    }

    #endregion

    public void LoadGame()
    {
        float[] playerScore = { 0 }; //TODO: remove references to old scoring system
        _playerInfo.CopyPlayerData(_ballType, colorSlider.value, playerScore);
        SceneManager.LoadScene(SceneManager.GetActiveScene().buildIndex + 1);
    }
}
```
</details>

<details>
<summary>The save manager script, which handles database functionality</summary>
  
  ```
using UnityEngine;
using Firebase;
using Firebase.Auth;
using Firebase.Database;
using Firebase.Extensions;

public class SaveManager : MonoBehaviour
{
    private PlayerInfo _playerInfo;
    private StartupScreenScript _startupScript;
    
    //Singleton variables
    private static SaveManager _instance;
    public static SaveManager Instance
    {
        get { return _instance; }
    }
    
    //Functions that gets called after load or save is completed
    public delegate void OnLoadedDelegate(string jsonString);
    public delegate void OnSaveDelegate();
    public delegate void OnRegistrationDelegate(bool anonymousRegistration, PlayerStats playerStats);
    public delegate void OnSigninDelegate(string jsonString);
    
    //Firebase namespaces
    private FirebaseAuth auth;
    private FirebaseDatabase db;

    //Key aliases
    const string FBKEY_USERS_PATH = "users";
    const string FBKEY_USERSDATA_PATH = "user_data";

    private void Start()
    {
        //Singleton setup
        if (_instance == null)
        {
            _instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(this.gameObject);
        }
        
        FirebaseApp.CheckAndFixDependenciesAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogError(task.Exception);
        
            auth = FirebaseAuth.DefaultInstance;
            db = FirebaseDatabase.DefaultInstance;
            db.SetPersistenceEnabled(false);
        });
        
        _playerInfo = FindObjectOfType<PlayerInfo>();
        _startupScript = FindObjectOfType<StartupScreenScript>();
    }

    #region USER REGISTRATION FUNCTIONS

    ////////////////////////////// USER REGISTRATION FUNCTIONS //////////////////////////////
    
    public void RegisterNewUser(string email, string password, OnRegistrationDelegate onRegistrationDelegate)
    {
        Debug.Log("Starting user registration");

        PlayerStats playerStats = new PlayerStats();

        string jsonString = JsonUtility.ToJson(playerStats);
        
        auth.CreateUserWithEmailAndPasswordAsync(email, password).ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogError(task.Exception);
            else
            {
                FirebaseUser newUser = task.Result;
                _playerInfo.userID = newUser.UserId;
                _playerInfo.playerEmail = newUser.Email;
                _playerInfo.playerIsAnonymous = false;
                Debug.LogFormat("local data saved: {0}, {1}, {2}", _playerInfo.userID, _playerInfo.playerEmail, _playerInfo.playerIsAnonymous.ToString());
                Debug.LogFormat("User registered: {0}, user ID: {1}", _playerInfo.playerEmail, _playerInfo.userID);
                
                Debug.LogFormat("Attempting to start data save: {0}", jsonString);
                db.RootReference.Child(FBKEY_USERS_PATH).Child(newUser.UserId).Child(FBKEY_USERSDATA_PATH)
                    .SetRawJsonValueAsync(jsonString);
                Debug.Log("User profile data created");

                onRegistrationDelegate.Invoke(false, playerStats);
            }
        });
    }
    
    public void RegisterUserName(string userName, OnSaveDelegate onSaveDelegate)
    {
        UserProfile newUserProfile = new UserProfile
        {
            DisplayName = userName
        };

        auth.CurrentUser.UpdateUserProfileAsync(newUserProfile).ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogError(task.Exception);
            else
            {
                _playerInfo.playerID = auth.CurrentUser.DisplayName;
                _playerInfo.playerName = auth.CurrentUser.DisplayName;
                onSaveDelegate.Invoke();
            }
        });
    }

    public void AnonymousRegistrationIn(string playerName, OnRegistrationDelegate onRegistrationDelegate)
    {
        PlayerStats playerStats = new PlayerStats();
        
        auth.SignInAnonymouslyAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogError(task.Exception);
            else
            {
                FirebaseUser newUser = task.Result;
                Debug.LogFormat("User signed in anonymously: {0}, user ID: {1}", playerName, newUser.UserId);
                _playerInfo.playerName = playerName;
                _playerInfo.playerIsAnonymous = true;
                onRegistrationDelegate.Invoke(true, playerStats);
            }
        });
    }
    
    #endregion

    #region USER SIGN-IN FUNCTIONS
    ////////////////////////////// USER SIGN-IN FUNCTIONS //////////////////////////////
    public void UserSignIn(string email, string password, OnSigninDelegate onSigninDelegate)
    {
        auth.SignInWithEmailAndPasswordAsync(email, password).ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogError(task.Exception);
            else
            {
                FirebaseUser newUser = task.Result;
                Debug.LogFormat("User signed in: {0}, user ID: {1}", newUser.DisplayName, newUser.UserId);
                _playerInfo.playerID = newUser.DisplayName;
                _playerInfo.playerName = _playerInfo.playerID;
                _playerInfo.userID = newUser.UserId;
                _playerInfo.playerEmail = newUser.Email;
                _playerInfo.playerIsAnonymous = false;
                
                db.RootReference.Child(FBKEY_USERS_PATH).Child(newUser.UserId).Child(FBKEY_USERSDATA_PATH)
                    .GetValueAsync().ContinueWithOnMainThread(task1 =>
                {
                    if (task.Exception != null)
                        Debug.LogWarning(task1.Exception);
                    else
                    {
                        string jsonString = task1.Result.GetRawJsonValue();
                        onSigninDelegate.Invoke(jsonString);
                    }
                });
            }
        });
    }

    #endregion

    #region Save/Load operations
    ////////////////////////////// SAVE/LOAD OPERATIONS FUNCTIONS //////////////////////////////
    public void SaveToFirebase(PlayerStats playerStats)
    {
        string jsonString = JsonUtility.ToJson(playerStats);
        
        db.RootReference.Child(FBKEY_USERS_PATH).Child(auth.CurrentUser.UserId).Child(FBKEY_USERSDATA_PATH)
            .SetRawJsonValueAsync(jsonString).ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogWarning(task.Exception);
        });
    }

    public void LoadFromFirebase(OnLoadedDelegate onLoadedDelegate)
    {
        db.RootReference.Child(FBKEY_USERS_PATH).Child(auth.CurrentUser.UserId).Child(FBKEY_USERSDATA_PATH)
            .GetValueAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogError(task.Exception);
            else
            {
                string jsonString = task.Result.GetRawJsonValue();
                onLoadedDelegate.Invoke(jsonString);
                _startupScript.SignInCallback(jsonString);
            }

        });
    }
    #endregion
    
    
    public void RemoveDataFromFirebase(string path)
    {
        db.RootReference.Child(path).RemoveValueAsync();
    }
}

  ```
</details>

<details>
  <summary>The high score manager, which adds and retrieves the high scores to and from the database</summary>

  ```
using System;
using System.Collections.Generic;
using System.Linq;
using Firebase.Auth;
using Firebase.Database;
using Firebase.Extensions;
using TMPro;
using UnityEngine;

public class HighScoreManager : MonoBehaviour
{
    [SerializeField] private GameObject highScorePanel;
    [SerializeField] private TMP_Text scoreDisplay;
    [SerializeField] private GameObject highScoreEntriesPanel;
    
    private const string FBKEY_SCORES_PATH = "scores";
    private const string FBKEY_SCOREENTRY_PATH = "entry";
    public int currentScore;
    private FirebaseDatabase db;
    private PlayerInfo playerInfo;
    private HighScoreEntry[] scoreEntries;
    private bool scoreBoardLoaded;
    private List<GameObject> highScoreListObjects = new List<GameObject>();

    public delegate void onScoreFetchedDelegate(List<HighScoreEntry> scoreList);
    
    private void Start()
    {
        db = FirebaseDatabase.DefaultInstance;
        playerInfo = FindObjectOfType<PlayerInfo>();
    }

    public void LoadScoreBoard()
    {
        FetchScoreBoard(ShowScoreBoard);
    }

    public void ShowScoreBoard(List<HighScoreEntry> scoreList)
    {
        scoreList.Sort((x, y) => y.score.CompareTo(x.score));
        foreach (var item in scoreList.Take(10))
        {
            GameObject thisScorePanel = Instantiate(highScorePanel, highScoreEntriesPanel.transform);
            HighScorePanelScript panelScript = thisScorePanel.GetComponent<HighScorePanelScript>();
            panelScript.updateFields(item.name, item.score, item.Date);
            highScoreListObjects.Add(thisScorePanel);
        }
    }

    public void ClearScoreBoard()
    {
        foreach (GameObject item in highScoreListObjects)
        {
            Destroy(item);
        }
    }
    
    public void FetchScoreBoard(onScoreFetchedDelegate onScoreFetchedDelegate)
    {
        db.RootReference.Child(FBKEY_SCORES_PATH).GetValueAsync().ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogWarning(task.Exception);
            
            List<HighScoreEntry> scoreList = new List<HighScoreEntry>();

            int scoreListPosition = 0;
            
            foreach (var item in task.Result.Children)
            {
                string jsonString = (string)item.Value;
                scoreList.Insert(scoreListPosition,  JsonUtility.FromJson<HighScoreEntry>(jsonString));
                scoreListPosition++;
            }
            
            onScoreFetchedDelegate(scoreList);
            });
        }
    
    public void UpdateCurrentScore(int incomingScoreValue)
    {
        currentScore += incomingScoreValue;
        scoreDisplay.text = currentScore.ToString();
    }

    public void AddScoreToFirebase()
    {
        HighScoreEntry highScoreEntry = new HighScoreEntry
        {
            UserID = FirebaseAuth.DefaultInstance.CurrentUser.UserId,
            name = playerInfo.playerName,
            score = currentScore,
            Date = DateTime.Now.ToString()
        };

        if (highScoreEntry.name == "")
            highScoreEntry.name = "default";
        
        
        string jsonString = JsonUtility.ToJson(highScoreEntry);
        
        db.RootReference.Child(FBKEY_SCORES_PATH).Child(FBKEY_SCOREENTRY_PATH + "_" + highScoreEntry.name + "_" + DateTime.Now).SetValueAsync(jsonString).ContinueWithOnMainThread(task =>
        {
            if (task.Exception != null)
                Debug.LogWarning(task.Exception);
        });
    }
}
```
</details>

<details>
  <summary>The high scores are contained in objects that are converted to JSon</summary>

```
[Serializable]
public class HighScoreEntry
{
    public string UserID;
    public string name = "";
    public int score;
    public string Date;
}
```
</details>

### Gravity-based gameplay
The actual gameplay in the game is as simple as can be. Since the ball is rolling down on a board, all you have to do is tilt your phone to control how it rolls. The script for handling this simply takes the input from the gyro and then updates the gravity in the game's physics engine.

<details>
  <summary>The gyroinput script</summary>

  ```
using System;
using UnityEngine;

public class GyroInput : MonoBehaviour
{
    private Rigidbody rgbd;

    void Start()
    {
        Input.gyro.enabled = true;
        rgbd = gameObject.GetComponent<Rigidbody>();
    }

    void FixedUpdate()
    {
        if (Time.timeScale > 0)
            GravityChange();
    }

    private void GravityChange()
    {
        float gravityX = Input.acceleration.x * 9.81f;
        float gravityY = Input.acceleration.y * 9.81f;
        
        gravityX = Math.Clamp(gravityX, -7f, 7f);
        gravityY = Math.Clamp(gravityY, -11f, -4f);
        
        Physics.gravity = new Vector3(gravityX, gravityY, 0);
        
        rgbd.AddForce(Physics.gravity * (2 * rgbd.mass));
    }
}
```

</details>

## Summary
While this game is short and was mainly just to explore online functionality and mobile development, it was still fun to experiement a bit with a new type of game control.
