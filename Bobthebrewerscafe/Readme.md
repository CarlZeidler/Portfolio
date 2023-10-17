# Bob the Brewer's Café

![BobthebrewerscafeTItle](/Assets/BobthebrewerscafeTItle.png)

Bob the Brewer's Café is a café management simulator where you work in Bob's café to make boba tea for customers before they get angry and leave. You as the player are responsible for brewing and serving the tea as well as cleaning up dishes and collecting payment.<br>
## Main contributions
### Guest behavior
For a café to function it obviously needs to have guests. I created a state machine to control how the guests behaves when visiting the player's establishment. The state machine utilizes an Enum and creates the various states as objects which, depending on which state is currently assigned using an enum, limits how the player is able to interact with the guests.

![image](https://github.com/CarlZeidler/Portfolio/assets/113012261/dee5ccd6-21cc-49ab-8256-c8a48a676038)

![Unity_Z6yCqPM3pO](https://github.com/CarlZeidler/Portfolio/assets/113012261/a2a36a9b-1edc-4b16-89bb-90aefec0e49d)

<br>
The state machine indexes the states in an array and uses it as reference when accessing the functionality in the state. Each frame it runs the update function in the currently active state.
<details>
  <summary>The state machine itself</summary>

  ```
public class GuestStatemachine
{
    public GuestState[] states;
    public Guest guest;
    public GuestStateID currentState;

    public GuestStatemachine(Guest guest)
    {
        this.guest = guest;
        int numstates = System.Enum.GetNames(typeof(GuestStateID)).Length;
        states = new GuestState[numstates];
    }

    public void RegisterState(GuestState state)
    {
        int index = (int)state.GetID();
        states[index] = state;
        
        //Used to register states in the state machine so that they can be entered
    }

    public GuestState GetState(GuestStateID stateId)
    {
        int index = (int)stateId;
        return states[index];
        
        //Returns what state the guest is currently in, used to determine which state should be ran
    }
    
    public void Update()
    {
        GetState(currentState)?.Update(guest);
        
        //Runs the "Update" function in the current state's script
    }

    public void ChangeState(GuestStateID newState)
    {
        GetState(currentState)?.Exit(guest);
        currentState = newState;
        GetState(currentState)?.Enter(guest);
        
        //Changes what state the guest is in. First runs the "Exit" function in the current state, then changes
        //state and runs the "Enter" function in the new state.
    }

    public void SetInitialState(GuestStateID newState)
    {
        currentState = newState;
        GetState(currentState)?.Enter(guest);
        
        //Sets the initial state for the guest. Should always be "Arriving."
    }
}

public enum GuestStateID
{
    Arrived,
    AtDoor,
    AtTable,
    Ordered,
    Served,
    Leaving,
    Angry
}

public interface GuestState
{
    GuestStateID GetID();
    void Enter(Guest guest);
    void Update(Guest guest);
    void Exit(Guest guest);
}

```
</details>

<details>
  <summary>The guest script's implementation of the state machine</summary>

  ```
    void Start()
    {
     
        //Create references for use in state machine state scripts
        navMeshAgent = GetComponent<NavMeshAgent>();
        orderImg = FindObjectOfType<OrderImageUI>();
        camera = Camera.main;
        
        guestCanvas.worldCamera = camera;
        
        //Register state machine and state machine states
        stateMachine = new GuestStatemachine(this);
        stateMachine.RegisterState(new GuestArrivedState());
        stateMachine.RegisterState(new GuestAtDoorState());
        stateMachine.RegisterState(new GuestAtTableState());
        stateMachine.RegisterState(new GuestOrderedState());
        stateMachine.RegisterState(new GuestServedState());
        stateMachine.RegisterState(new GuestLeavingState());
        stateMachine.RegisterState(new GuestAngryState());
        //Sets the initial state of the guest, should always be "Arriving"
        stateMachine.SetInitialState(initialState);
    }
...
    void Update()
    {
        stateMachine.Update();
    }
```
</details>

<details>
  <summary>Sample state: Arrived</summary>

  ```
using UnityEngine;

public class GuestArrivedState : GuestState
{
    private bool moving;
    
    public GuestStateID GetID()
    {
        return GuestStateID.Arrived;
    }

    public void Enter(Guest guest)
    {
        MoveToDestination(guest, guest.door.doorEnterSpot);
    }

    public void Update(Guest guest)
    {
        CheckIfAtDestination(guest);
    }
    
    public void Exit(Guest guest)
    {
        
    }
    
    private void MoveToDestination(Guest guest, Transform target)
    {
        guest.navMeshAgent.destination = target.position;
        moving = true;
        
        //Moves the guest to a new position via navmesh system using a transform component reference.
    }
    
    private void CheckIfAtDestination(Guest guest)
    {
        if (moving && ReachedDestinationOrGaveUp(guest))
        {
                guest.navMeshAgent.ResetPath();
                moving = !moving;
                guest.stateMachine.ChangeState(GuestStateID.AtDoor);
        }
    }
    
    public bool ReachedDestinationOrGaveUp(Guest guest)
    {
        if (!guest.navMeshAgent.pathPending)
        {
            if (guest.navMeshAgent.remainingDistance <= guest.navMeshAgent.stoppingDistance)
            {
                if (!guest.navMeshAgent.hasPath || guest.navMeshAgent.velocity.sqrMagnitude == 0f)
                {
                    return true;
                }
            }
        }
        return false;
        //This function will check if the guest has reached their destination.
    }
}
```
</details>

<details>
  <summary>Sample state: Angry</summary>

  ```
using UnityEngine;

public class GuestAngryState : GuestState
{
    private bool moving;
    
    public GuestStateID GetID()
    {
        return GuestStateID.Angry;
    }

    public void Enter(Guest guest)
    {
        guest.orderText.color = guest.angryTextColor;
        guest.orderText.font = guest.gibberishFont;
        guest.orderText.SetText("Leaving! Grr!");
        MoveToDestination(guest, guest.door.doorExitSpot);
        GameManager.Instance.AddAngryGuest();
        guest.orderImg.RemoveOrderImage(guest.orderTicketRef);
        guest.teaOrderImg.gameObject.SetActive(false);
        guest.animator.SetBool("Moving", true);
    }

    public void Update(Guest guest)
    {
        if (guest.door.open)
            guest.stateMachine.ChangeState(GuestStateID.Leaving);
        CheckIfAtDestination(guest);
        
        guest.guestCanvas.transform.forward = guest.camera.transform.forward;
    }

    private void MoveToDestination(Guest guest, Transform target)
    {
        guest.navMeshAgent.destination = target.position;
        moving = true;
        
        //Moves the guest to a new position via navmesh system using a transform component reference.
    }
    
    private void CheckIfAtDestination(Guest guest)
    {
        if (moving && ReachedDestinationOrGaveUp(guest))
        {
            guest.navMeshAgent.ResetPath();
            moving = !moving;
            // guest.door.OpenDoor();
        }
    }
    
    public void Exit(Guest guest)
    {
        GameManager.Instance.freeSeats++;
        GameManager.Instance.ReturnFreeSeat(guest.chairRef);
        guest.chairRef.tableRef.guestRef = null;
        guest.guestInteraction.ToggleAngerMeter(false);
    }
    
    public bool ReachedDestinationOrGaveUp(Guest guest)
    {
        if (!guest.navMeshAgent.pathPending)
        {
            if (guest.navMeshAgent.remainingDistance <= guest.navMeshAgent.stoppingDistance)
            {
                if (!guest.navMeshAgent.hasPath || guest.navMeshAgent.velocity.sqrMagnitude == 0f)
                {
                    return true;
                }
            }
        }
        return false;
        //This function will check if the guest has reached their destination.
    }
}
```
</details>

<details>
  <summary>Sample state: Ordered</summary>
   
  ```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class GuestOrderedState : GuestState
{
    public GuestStateID GetID()
    {
        return GuestStateID.Ordered;
    }

    public void Enter(Guest guest)
    {
        AddOrder(guest);
        guest.orderTextFrameElement.SetActive(false);
    }

    public void Update(Guest guest)
    {
        guest.guestCanvas.transform.forward = guest.camera.transform.forward;
    }

    public void Exit(Guest guest)
    {
        guest.orderTextFrameElement.SetActive(true);
    }

    private void AddOrder(Guest guest)
    {
        
        guest.orderImg.SpawnOrderImage(guest.teaType, guest);
        guest.teaOrderImg.gameObject.SetActive(true);
        if (guest.teaType is TeaType.TypeA)
        {
            guest.teaOrderImg.GetComponent<Image>().sprite = guest.teaOrderImg1;
        }
        else if (guest.teaType is TeaType.TypeB)
        {
            guest.teaOrderImg.GetComponent<Image>().sprite = guest.teaOrderImg2;
        }
        guest.orderText.SetText(" ");
    }
}
```
</details>

### Seat assignation
The café has a number of possible seats that the guests can use when visiting. In order to keep track of seats and who sat where and randomize which seat guests picked I implemented seat assignation functionality within the gamemanager that "hands out" seats when guests requests them. The guest then "returns" the seat when they leave the café.
<details>
  <summary>The seat assignation functionality</summary>
  
  ```
  public List<ISeat> freeSeatsInScene = new();
    //Maintains a list of seats that are in the scene

    public int freeSeats = 0;
    //Shows how many free seats in the scene, added to by the Seats upon play start.
...
public Chair AssignSeat()
    {
        System.Random rand = new System.Random();
        for (int i = freeSeatsInScene.Count - 1; i > 0; i--)
        {
            int j = rand.Next(i + 1);
            ISeat temp = freeSeatsInScene[i];
            freeSeatsInScene[i] = freeSeatsInScene[j];
            freeSeatsInScene[j] = temp;
        }

        foreach (ISeat seat in freeSeatsInScene)
        {
            if (!seat.HasDirtyDish())
            {
                Chair chosenRef = seat.GetChairRef();
                freeSeatsInScene.Remove(seat);
                freeSeats--;
                return chosenRef;
            }
        }

        Debug.Log("If you see this, something has gone wrong in the AssignSeat function in the Game Manager");
        return null;

        //Gives a reference to a free seat to a guest upon request.
    }
...
 public void ReturnFreeSeat(Chair chair)
    {
        freeSeatsInScene.Add(chair);
    }
  ```
</details>

### New interaction system
As the main way the player interacts with the game is by picking up things or taking orders from the guests, a robust interaction system was needed. The project group initially designed one but as the project grew it became necessary to revamp it to make it more consistent. I did this by implementing a new abstract class that all interactable objects inherited from, which was then checked towards by
the new interaction system script.

![Unity_eelmzCOkgb](https://github.com/CarlZeidler/Portfolio/assets/113012261/37003f37-9b7e-423e-a03e-af125ec9d343)

<details>
  <summary>The new interaction system that is on the player game object</summary>

  ```
using System.Collections.Generic;
using System.Linq;
using Carl.NewInteractionSystem;
using UnityEngine;
using TMPro;

public class NewInteract : MonoBehaviour
{
    [SerializeField] private TMP_Text toolTipDisplay;
    [SerializeField] private Vector3 offset;
    
    public List<NewAbstractInteractable> interactables = new();
    public NewAbstractInteractable heldObjectRef;
    private Camera mainCameraRef;

    private void Start()
    {
        mainCameraRef = Camera.main;
        Invoke(nameof(UpdatePlayerToolTip), .5f);
    }

    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.E))
        {
            NewAbstractInteractable closest;
            
            if (interactables.Count != 0 && !heldObjectRef)
            {
                closest = FindClosestInteractable();
                if (closest)
                    closest.Interact(this);
            }
            else if (interactables.Count != 0 && heldObjectRef)
            {
                heldObjectRef.Interact(this);
            }
        }
    }

    private void UpdatePlayerToolTip()
    {
        if (heldObjectRef)
        {
            switch (heldObjectRef.newItemType)
            {
                case NewItemType.boba:
                    toolTipDisplay.SetText("E: Throw boba");
                    break;
                case NewItemType.fullBucket:
                    Interactable_NewBucket bucketScript = heldObjectRef.GetComponent<Interactable_NewBucket>();
                    if (bucketScript.hasWater)
                    {
                        if (bucketScript.closeToPot)
                        {
                            toolTipDisplay.SetText("E: Add water to pot");
                        }
                        else
                        {
                            toolTipDisplay.SetText("E: Drop");
                        }
                    }
                    else if (!bucketScript.hasWater)
                    {
                        if (bucketScript.closeToWater)
                        {
                            toolTipDisplay.SetText("E: Draw water");
                        }
                        else
                        {
                            toolTipDisplay.SetText("E: Drop");
                        }
                    }
                    break;
                case NewItemType.finishedTea:
                    Interactable_NewFullTea fullTeaScript = heldObjectRef.GetComponent<Interactable_NewFullTea>();
                    NewBobaTeaHandler closestTable = fullTeaScript.FindClosestTable();
                    if (closestTable)
                    {
                        if (closestTable.guestRef)
                        {
                            if (closestTable.guestRef.stateMachine.currentState is GuestStateID.Ordered)
                            {
                                toolTipDisplay.SetText("E: Serve tea");
                            }
                            else if (closestTable.guestRef.stateMachine.currentState is GuestStateID.AtTable)
                            {
                                toolTipDisplay.SetText("E: Take order");
                            }
                        }
                    }
                    else
                    {
                        toolTipDisplay.SetText("E: Drop");
                    }
                    break;
                case NewItemType.dirtyTea:
                    toolTipDisplay.SetText("E: Throw dirty dish");
                    break;
            }
        }
        else
        {
            NewAbstractInteractable closest = FindClosestInteractable();

            if (!closest)
            {
                toolTipDisplay.SetText("");
            }
            else
            {
                switch (closest.newItemType)
                {
                    case NewItemType.boba:
                        toolTipDisplay.SetText("E: Pick up boba pearl");
                        break;
                    case NewItemType.fullBucket:
                        Interactable_NewBucket bucketScript = closest.GetComponent<Interactable_NewBucket>();
                        if (bucketScript.hasWater)
                            toolTipDisplay.SetText("E: Pick up full bucket");
                        else if (!bucketScript.hasWater)
                            toolTipDisplay.SetText("E: Pick up empty bucket");
                        break;
                    case NewItemType.finishedTea:
                        toolTipDisplay.SetText("E: Pick up tea");
                        break;
                    case NewItemType.dirtyTea:
                        toolTipDisplay.SetText("E: Pick up dirty dishes");
                        break;
                    case NewItemType.bobaHandler:
                        NewBobaTeaHandler bobaHandlerRef = closest.GetComponent<NewBobaTeaHandler>();
                        if (bobaHandlerRef.guestRef.stateMachine.currentState is GuestStateID.AtTable)
                            toolTipDisplay.SetText("E: Take order");
                        else
                            toolTipDisplay.SetText("");
                        break;
                }
            }
        }
        
        Invoke(nameof(UpdatePlayerToolTip), .3f);
    }
    
    
    private NewAbstractInteractable FindClosestInteractable()
    {
        CleanList();
        if (interactables.Count == 0)
            return null;
        NewAbstractInteractable closest = interactables.OrderBy(x => 
                Vector3.Distance(transform.position, x.transform.position)).First();
        return closest;
    }

    private void CleanList()
    {
        for (int i = interactables.Count - 1; i >= 0; i--)
        {
            if (interactables[i] is null)
            {
                interactables.RemoveAt(i);
            }
            else if (interactables[i].newItemType is NewItemType.bobaHandler &&
                     !interactables[i].GetComponent<NewBobaTeaHandler>().guestRef)
            {
                interactables.RemoveAt(i);
            }
        }        
    }
    
    public void HoldingSomething(NewAbstractInteractable newInteractable)
    {
        heldObjectRef = newInteractable;
    }

    public void NoLongerHoldingSomething()
    {
        heldObjectRef = null;
    }

    private void OnTriggerEnter(Collider other)
    {
        if (other.GetComponent<NewAbstractInteractable>() != null)
        {
            interactables.Add(other.GetComponent<NewAbstractInteractable>());
        }
    }

    private void OnTriggerExit(Collider other)
    {
        if (other.GetComponent<NewAbstractInteractable>() != null)
        {
            interactables.Remove(other.GetComponent<NewAbstractInteractable>());
        }
    }
}
```
</details>

<details>
  <summary>The new abstract class</summary>

  ```
  using UnityEngine;

namespace Carl.NewInteractionSystem
{
    public enum NewItemType
    {
        emptyBucket,
        fullBucket,
        boba,
        finishedTea,
        dirtyTea,
        cleanTea,
        bobaHandler
    }

    public abstract class NewAbstractInteractable : MonoBehaviour
    {
        [SerializeField] public NewItemType newItemType;
        public bool isHeld;
        
        public NewInteract playerInteractRef;
        public Transform toolParent;
        
        public abstract void Interact(NewInteract newInteract);
        
        public void Hold(NewInteract newInteract)
        {
            //Connect holdable to player model
            toolParent = GameObject.Find("ToolParent").transform;
            gameObject.GetComponent<Rigidbody>().velocity = Vector3.zero;
            gameObject.GetComponent<Rigidbody>().isKinematic = true;
            
            gameObject.transform.position = toolParent.transform.position;
            gameObject.transform.rotation = toolParent.transform.rotation;
            
            
            gameObject.GetComponent<Collider>().enabled = false;

            gameObject.transform.SetParent(toolParent);

            isHeld = true;

            newInteract.HoldingSomething(this);
            newInteract.interactables.Remove(this);
        }

        public void Drop(NewInteract newInteract)
        {
            //Detach holdable from player model
            toolParent.DetachChildren();
            gameObject.transform.eulerAngles = new Vector3(gameObject.transform.position.x, gameObject.transform.position.z, gameObject.transform.position.y);
            gameObject.GetComponent<Rigidbody>().isKinematic = false;
            gameObject.GetComponent<MeshCollider>().enabled = true;

            isHeld = false;
        
            playerInteractRef.NoLongerHoldingSomething();
        }

        public abstract void Throw(NewInteract newInteract);
        //If held item is throwable, throw it

        public abstract void TeaOperations(NewInteract newInteract);
        //Serve customer if possible

        public abstract void WaterOperations(NewInteract newInteract);
        //Water operations
    }
}
```
</details>

<details>
  <summary>Sample interactable object that inherits from the abstract class</summary>
  
  ```
using System.Collections;
using System.Collections.Generic;
using Carl.NewInteractionSystem;
using UnityEngine;

public class Interactable_NewBoba : NewAbstractInteractable
{
    Vector3 startThrowPos;

    public Transform target;
    public Transform posOverHead;

    private float t = 0;

    public bool isBallFlying;
    bool bobaIsStolen;

    SoundManager soundManager;

    private void Start()
    {
        target = GameObject.Find("TargetPoint Boba shooter").transform;
        posOverHead = GameObject.Find("PosOverHead").transform;
        soundManager = FindFirstObjectByType<SoundManager>();
    }

    private void Update()
    {
        Throw(playerInteractRef);

    }

    public override void Interact(NewInteract newInteract)
    {
        playerInteractRef = newInteract;

        if (isHeld)
        {
            startThrowPos = posOverHead.position;
            isBallFlying = true;
            t = 0;
            Throw(playerInteractRef);
            isHeld = false;
            soundManager.BobaThrow();
            gameObject.GetComponent<BobaMovement>().enabled = true;
        }
        else
        {
            gameObject.GetComponent<BobaMovement>().enabled = false;
            Hold(playerInteractRef);
        }

    }

    public override void Throw(NewInteract newInteract)
    {
        if (isBallFlying)
        {
            // bobashooter script logic
            gameObject.GetComponent<SphereCollider>().enabled = false;

            t += Time.deltaTime;
            toolParent.DetachChildren();

            //Implement throwing functionality
            float duration = 1.5f;
            float t01 = t / duration;

            // move to target
            Vector3 A = startThrowPos;
            Vector3 B = target.position;
            Vector3 pos = Vector3.Lerp(A, B, t01);

            // move in arc
            Vector3 arc = Vector3.up * 5 * Mathf.Sin(t01 * 3.14f);
            gameObject.transform.position = pos + arc;

            if (t01 >= 1)
            {
                gameObject.GetComponent<Rigidbody>().isKinematic = false;
                isBallFlying = false;

            }

            //Ensure the below function call is included
            playerInteractRef.NoLongerHoldingSomething();
        }
    }

    public override void TeaOperations(NewInteract newInteract)
    {
        //Not servable
    }

    public override void WaterOperations(NewInteract newInteract)
    {
        //Not a bucket
    }

    private void OnTriggerStay(Collider other)
    {

        if (other.gameObject.CompareTag("BrewingPot"))
        {
            gameObject.SetActive(false);
            gameObject.GetComponent<SphereCollider>().enabled = false;
            isBallFlying = false;
        }
    }

    private void OnTriggerEnter(Collider other)
    {

        if (other.gameObject.CompareTag("BrewingPot"))
        {
            isBallFlying = false;
            gameObject.GetComponent<SphereCollider>().enabled = false;
        }
    }
}
```
</details>

## Summary
This project let me explore and implment new programming patterns, and also features practical implementations of abstract classes and interfaces, which was very fun to explore.
