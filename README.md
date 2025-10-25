using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;

public class TouchJoystick : MonoBehaviour, IDragHandler, IPointerUpHandler, IPointerDownHandler
{
    public RectTransform background;
    public RectTransform handle;
    private Vector2 input = Vector2.zero;
    private float handleLimit;

    void Start()
    {
        if (background == null) background = GetComponent<RectTransform>();
        handleLimit = background.sizeDelta.x * 0.5f;
    }

    public void OnDrag(PointerEventData eventData)
    {
        Vector2 pos;
        RectTransformUtility.ScreenPointToLocalPointInRectangle(background, eventData.position, eventData.pressEventCamera, out pos);
        pos = Vector2.ClampMagnitude(pos, handleLimit);
        handle.anchoredPosition = pos;
        input = pos / handleLimit;
    }

    public void OnPointerDown(PointerEventData eventData) => OnDrag(eventData);

    public void OnPointerUp(PointerEventData eventData)
    {
        handle.anchoredPosition = Vector2.zero;
        input = Vector2.zero;
    }

    public float Horizontal => input.x;
    public float Vertical => input.y;
    public bool IsPressed => input.sqrMagnitude > 0.01f;
}using UnityEngine;

[RequireComponent(typeof(Rigidbody))]
public class PlayerController : MonoBehaviour
{
    public float moveSpeed = 4f;
    public float jumpForce = 5f;
    public TouchJoystick joystick;
    private Rigidbody rb;

    void Start()
    {
        rb = GetComponent<Rigidbody>();
        rb.constraints = RigidbodyConstraints.FreezeRotation;
    }

    void Update()
    {
        Vector3 dir = new Vector3(joystick.Horizontal, 0, joystick.Vertical);
        if (dir.magnitude > 0.1f)
        {
            Vector3 move = dir.normalized * moveSpeed;
            Vector3 worldMove = Camera.main.transform.TransformDirection(move);
            worldMove.y = 0;
            rb.velocity = new Vector3(worldMove.x, rb.velocity.y, worldMove.z);
            transform.forward = Vector3.Lerp(transform.forward, new Vector3(worldMove.x, 0, worldMove.z), Time.deltaTime * 10f);
        }
        else
        {
            // y-xz sabitle
            rb.velocity = new Vector3(0, rb.velocity.y, 0);
        }

        // Jump: UI'de bir buton bağla -> Input yönetimi
        if (Input.GetButtonDown("Jump"))
        {
            if (Mathf.Abs(rb.velocity.y) < 0.05f)
                rb.AddForce(Vector3.up * jumpForce, ForceMode.VelocityChange);
        }
    }
}using UnityEngine;
using UnityEngine.EventSystems;

public class StudioManager : MonoBehaviour
{
    public Camera editorCamera;
    public GameObject[] availablePrefabs; // atayacağın prefab listesi
    private GameObject previewInstance;
    private int selectedIndex = 0;

    void Update()
    {
        if (EventSystem.current.IsPointerOverGameObject(0)) return; // UI üzerine tıklamayla engelle

        if (Input.touchCount > 0)
        {
            Touch t = Input.GetTouch(0);
            if (t.phase == TouchPhase.Began)
            {
                Ray ray = editorCamera.ScreenPointToRay(t.position);
                if (Physics.Raycast(ray, out RaycastHit hit))
                {
                    // Eğer sahnede varsa ve "placer" modundaysan spawnla
                    Vector3 spawnPos = hit.point;
                    SpawnPrefab(spawnPos);
                }
            }
        }
    }

    public void SelectPrefab(int index)
    {
        selectedIndex = Mathf.Clamp(index, 0, availablePrefabs.Length - 1);
        UpdatePreview();
    }

    private void UpdatePreview()
    {
        if (previewInstance != null) Destroy(previewInstance);
        previewInstance = Instantiate(availablePrefabs[selectedIndex]);
        var cols = previewInstance.GetComponentsInChildren<Collider>();
        foreach (var c in cols) c.enabled = false;
        previewInstance.AddComponent<PreviewMarker>();
    }

    private void SpawnPrefab(Vector3 pos)
    {
        var go = Instantiate(availablePrefabs[selectedIndex], pos, Quaternion.identity);
        // Publish için temporary local ID/metadata at
        // örn: go.GetComponent<UniqueId>().Set(...);
    }
}
public class PreviewMarker : MonoBehaviour { } // sadece preview işaretçisiusing Photon.Pun;
using Photon.Realtime;
using UnityEngine;
using UnityEngine.SceneManagement;

public class PhotonManager : MonoBehaviourPunCallbacks
{
    public static PhotonManager Instance;

    void Awake() { Instance = this; DontDestroyOnLoad(gameObject); }

    void Start() { Connect(); }

    public void Connect()
    {
        if (!PhotonNetwork.IsConnected) PhotonNetwork.ConnectUsingSettings();
    }

    public override void OnConnectedToMaster()
    {
        PhotonNetwork.JoinLobby();
    }

    public void CreateRoom(string roomName)
    {
        RoomOptions options = new RoomOptions() { MaxPlayers = 12, IsVisible = true };
        PhotonNetwork.CreateRoom(roomName, options);
    }

    public void JoinRandomRoom()
    {
        PhotonNetwork.JoinRandomRoom();
    }

    public override void OnJoinRandomFailed(short returnCode, string message)
    {
        CreateRoom("Room_" + Random.Range(1000, 9999));
    }

    public override void OnJoinedRoom()
    {
        // Odaya girince oyun sahnesini yükle
        PhotonNetwork.LoadLevel("Prototype"); // Build Settings'e ekli olmalı
    }
}
