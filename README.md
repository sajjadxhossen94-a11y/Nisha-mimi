// GameManager.cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.SceneManagement;

public enum GameMode { FreeForAll, TeamDeathmatch, Survival, Training }
public enum MatchStatus { Lobby, InProgress, Ended }

public class GameManager : MonoBehaviour
{
    public static GameManager Instance { get; private set; }
    
    [Header("Game Settings")]
    [SerializeField] private int _maxPlayers = 15;
    [SerializeField] private float _matchDuration = 600f; // 10 minutes
    [SerializeField] private int _killLimitFFA = 25;
    [SerializeField] private int _killLimitTDM = 50;
    
    [Header("References")]
    [SerializeField] private GameObject _playerPrefab;
    [SerializeField] private Transform[] _spawnPoints;
    [SerializeField] private List<MapData> _maps;
    
    // Game State
    private GameMode _currentGameMode;
    private MatchStatus _matchStatus;
    private float _matchTimer;
    private int _playersAlive;
    private Dictionary<int, PlayerData> _players = new();
    private Dictionary<Team, int> _teamScores = new();
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    public void StartMatch(GameMode mode, string mapName)
    {
        _currentGameMode = mode;
        _matchStatus = MatchStatus.InProgress;
        _matchTimer = _matchDuration;
        _playersAlive = 0;
        
        // Initialize teams
        if (mode == GameMode.TeamDeathmatch)
        {
            _teamScores[Team.Red] = 0;
            _teamScores[Team.Blue] = 0;
        }
        
        // Load map
        LoadMap(mapName);
        
        // Spawn players
        StartCoroutine(SpawnPlayers());
        
        // Start match timer
        StartCoroutine(MatchTimer());
    }
    
    private IEnumerator SpawnPlayers()
    {
        int playerCount = GetPlayerCount();
        
        for (int i = 0; i < playerCount; i++)
        {
            Transform spawnPoint = GetSpawnPoint();
            GameObject player = Instantiate(_playerPrefab, spawnPoint.position, spawnPoint.rotation);
            
            PlayerData playerData = new PlayerData
            {
                PlayerID = i,
                PlayerObject = player,
                Kills = 0,
                Deaths = 0,
                Score = 0,
                Team = GetTeamAssignment(i),
                IsAlive = true
            };
            
            _players.Add(i, playerData);
            _playersAlive++;
            
            // Assign player controller
            PlayerController controller = player.GetComponent<PlayerController>();
            controller.Initialize(playerData);
            
            yield return new WaitForSeconds(0.1f);
        }
    }
    
    private IEnumerator MatchTimer()
    {
        while (_matchTimer > 0 && _matchStatus == MatchStatus.InProgress)
        {
            _matchTimer -= Time.deltaTime;
            
            // Check win conditions
            CheckWinConditions();
            
            yield return null;
        }
        
        EndMatch();
    }
    
    private void CheckWinConditions()
    {
        switch (_currentGameMode)
        {
            case GameMode.FreeForAll:
                // Check kill limit
                foreach (var player in _players.Values)
                {
                    if (player.Kills >= _killLimitFFA)
                    {
                        EndMatch(player.PlayerID);
                        return;
                    }
                }
                
                // Check time limit
                if (_matchTimer <= 0)
                {
                    // Find player with most kills
                    int winnerID = -1;
                    int maxKills = -1;
                    
                    foreach (var player in _players.Values)
                    {
                        if (player.Kills > maxKills)
                        {
                            maxKills = player.Kills;
                            winnerID = player.PlayerID;
                        }
                    }
                    
                    EndMatch(winnerID);
                }
                break;
                
            case GameMode.TeamDeathmatch:
                // Check team score
                if (_teamScores[Team.Red] >= _killLimitTDM)
                {
                    EndMatch(Team.Red);
                }
                else if (_teamScores[Team.Blue] >= _killLimitTDM)
                {
                    EndMatch(Team.Blue);
                }
                
                // Check time limit
                if (_matchTimer <= 0)
                {
                    Team winningTeam = _teamScores[Team.Red] > _teamScores[Team.Blue] ? Team.Red : Team.Blue;
                    EndMatch(winningTeam);
                }
                break;
                
            case GameMode.Survival:
                // Last player standing wins
                if (_playersAlive <= 1)
                {
                    foreach (var player in _players.Values)
                    {
                        if (player.IsAlive)
                        {
                            EndMatch(player.PlayerID);
                            return;
                        }
                    }
                }
                break;
        }
    }
    
    public void OnPlayerDeath(int victimID, int killerID)
    {
        if (!_players.ContainsKey(victimID)) return;
        
        PlayerData victim = _players[victimID];
        victim.IsAlive = false;
        victim.Deaths++;
        _playersAlive--;
        
        if (_players.ContainsKey(killerID) && victimID != killerID)
        {
            PlayerData killer = _players[killerID];
            killer.Kills++;
            killer.Score += 100;
            
            // Team score
            if (_currentGameMode == GameMode.TeamDeathmatch)
            {
                _teamScores[killer.Team]++;
            }
        }
        
        // Update UI
        UIManager.Instance.UpdateScoreboard(_players, _teamScores);
        
        // Respawn logic
        if (_currentGameMode != GameMode.Survival)
        {
            StartCoroutine(RespawnPlayer(victimID, 3f));
        }
    }
    
    private IEnumerator RespawnPlayer(int playerID, float delay)
    {
        yield return new WaitForSeconds(delay);
        
        if (_players.ContainsKey(playerID))
        {
            PlayerData player = _players[playerID];
            Transform spawnPoint = GetSpawnPoint();
            player.PlayerObject.transform.position = spawnPoint.position;
            player.PlayerObject.transform.rotation = spawnPoint.rotation;
            
            // Reset player health and weapons
            PlayerController controller = player.PlayerObject.GetComponent<PlayerController>();
            controller.Respawn();
            
            player.IsAlive = true;
            _playersAlive++;
        }
    }
    
    private void EndMatch(object winner = null)
    {
        _matchStatus = MatchStatus.Ended;
        
        // Show results screen
        UIManager.Instance.ShowMatchResults(_players, winner);
        
        // Award XP
        AwardExperience();
        
        // Return to lobby after delay
        StartCoroutine(ReturnToLobby(10f));
    }
    
    private void AwardExperience()
    {
        foreach (var player in _players.Values)
        {
            int xpGained = CalculateXP(player);
            SaveManager.Instance.AddExperience(xpGained);
        }
    }
    
    private int CalculateXP(PlayerData player)
    {
        int baseXP = 100;
        int killXP = player.Kills * 25;
        int scoreXP = player.Score / 10;
        int winBonus = player.IsWinner ? 200 : 0;
        
        return baseXP + killXP + scoreXP + winBonus;
    }
    
    private Transform GetSpawnPoint()
    {
        // Find safe spawn point
        List<Transform> availableSpawns = new List<Transform>();
        
        foreach (Transform spawn in _spawnPoints)
        {
            // Check if spawn is safe (no enemies nearby)
            Collider[] colliders = Physics.OverlapSphere(spawn.position, 5f);
            bool isSafe = true;
            
            foreach (var collider in colliders)
            {
                if (collider.CompareTag("Player"))
                {
                    isSafe = false;
                    break;
                }
            }
            
            if (isSafe)
            {
                availableSpawns.Add(spawn);
            }
        }
        
        if (availableSpawns.Count > 0)
        {
            return availableSpawns[Random.Range(0, availableSpawns.Count)];
        }
        
        return _spawnPoints[Random.Range(0, _spawnPoints.Length)];
    }
    
    private Team GetTeamAssignment(int playerIndex)
    {
        if (_currentGameMode != GameMode.TeamDeathmatch)
            return Team.None;
            
        return playerIndex % 2 == 0 ? Team.Red : Team.Blue;
    }
    
    private void LoadMap(string mapName)
    {
        // Load map prefab
        MapData mapData = _maps.Find(m => m.MapName == mapName);
        if (mapData != null)
        {
            GameObject map = Instantiate(mapData.MapPrefab);
            _spawnPoints = mapData.SpawnPoints;
        }
    }
    
    public int GetPlayerCount()
    {
        return _currentGameMode switch
        {
            GameMode.FreeForAll => 8,
            GameMode.TeamDeathmatch => 10,
            GameMode.Survival => 15,
            GameMode.Training => 1,
            _ => 8
        };
    }
    
    private IEnumerator ReturnToLobby(float delay)
    {
        yield return new WaitForSeconds(delay);
        SceneManager.LoadScene("Lobby");
    }
}

[System.Serializable]
public class MapData
{
    public string MapName;
    public GameObject MapPrefab;
    public Transform[] SpawnPoints;
}

public class PlayerData
{
    public int PlayerID;
    public GameObject PlayerObject;
    public int Kills;
    public int Deaths;
    public int Score;
    public Team Team;
    public bool IsAlive;
    public bool IsWinner;
}

public enum Team { None, Red, Blue } 
// PlayerController.cs
using System.Collections;
using UnityEngine;

[RequireComponent(typeof(CharacterController), typeof(AudioSource))]
public class PlayerController : MonoBehaviour
{
    [Header("Movement Settings")]
    [SerializeField] private float _walkSpeed = 5f;
    [SerializeField] private float _runSpeed = 8f;
    [SerializeField] private float _jumpHeight = 2f;
    [SerializeField] private float _gravity = -20f;
    [SerializeField] private float _groundCheckDistance = 0.2f;
    [SerializeField] private LayerMask _groundMask;
    
    [Header("Mecha Settings")]
    [SerializeField] private MechaType _mechaType = MechaType.Medium;
    [SerializeField] private float _mechaHealth = 100f;
    [SerializeField] private float _mechaArmor = 50f;
    [SerializeField] private float _energyCapacity = 100f;
    [SerializeField] private float _energyRegenRate = 10f;
    
    [Header("References")]
    [SerializeField] private Transform _cameraTransform;
    [SerializeField] private Transform _weaponHolder;
    [SerializeField] private GameObject _deathEffect;
    [SerializeField] private AudioClip _jumpSound;
    [SerializeField] private AudioClip _landSound;
    
    // Components
    private CharacterController _controller;
    private AudioSource _audioSource;
    private PlayerData _playerData;
    private WeaponSystem _weaponSystem;
    
    // Movement Variables
    private Vector3 _velocity;
    private float _currentSpeed;
    private bool _isGrounded;
    private bool _isRunning;
    private bool _isCrouching;
    private bool _isJumping;
    
    // Mecha Variables
    private float _currentHealth;
    private float _currentArmor;
    private float _currentEnergy;
    private bool _isDead;
    
    // Input
    private Vector2 _moveInput;
    private Vector2 _lookInput;
    private bool _jumpPressed;
    private bool _runPressed;
    private bool _crouchPressed;
    
    public void Initialize(PlayerData playerData)
    {
        _playerData = playerData;
        _controller = GetComponent<CharacterController>();
        _audioSource = GetComponent<AudioSource>();
        _weaponSystem = GetComponent<WeaponSystem>();
        
        // Set mecha stats based on type
        SetMechaStats(_mechaType);
        
        // Reset state
        _currentHealth = _mechaHealth;
        _currentArmor = _mechaArmor;
        _currentEnergy = _energyCapacity;
        _isDead = false;
        
        // Initialize weapon system
        if (_weaponSystem != null)
        {
            _weaponSystem.Initialize(this);
        }
        
        // Set team color
        SetTeamColor(playerData.Team);
    }
    
    private void Update()
    {
        if (_isDead) return;
        
        HandleInput();
        HandleMovement();
        HandleRotation();
        HandleMechaAbilities();
        
        // Update UI
        UpdateUI();
    }
    
    private void HandleInput()
    {
        // Mobile/PC input abstraction
        _moveInput = InputManager.Instance.GetMovement();
        _lookInput = InputManager.Instance.GetLook();
        
        _jumpPressed = InputManager.Instance.GetButtonDown("Jump");
        _runPressed = InputManager.Instance.GetButton("Run");
        _crouchPressed = InputManager.Instance.GetButton("Crouch");
        
        // Weapon input
        bool firePressed = InputManager.Instance.GetButton("Fire");
        bool aimPressed = InputManager.Instance.GetButton("Aim");
        bool reloadPressed = InputManager.Instance.GetButtonDown("Reload");
        
        if (_weaponSystem != null)
        {
            if (firePressed) _weaponSystem.StartFiring();
            else _weaponSystem.StopFiring();
            
            _weaponSystem.SetAiming(aimPressed);
            
            if (reloadPressed) _weaponSystem.Reload();
        }
    }
    
    private void HandleMovement()
    {
        // Ground check
        _isGrounded = Physics.CheckSphere(transform.position, _groundCheckDistance, _groundMask);
        
        if (_isGrounded && _velocity.y < 0)
        {
            _velocity.y = -2f;
        }
        
        // Movement direction
        Vector3 moveDirection = transform.right * _moveInput.x + transform.forward * _moveInput.y;
        
        // Speed calculation
        _currentSpeed = _isRunning ? _runSpeed : _walkSpeed;
        
        // Apply crouch
        if (_crouchPressed && _isGrounded)
        {
            _isCrouching = true;
            _currentSpeed *= 0.5f;
            _controller.height = 1f;
        }
        else
        {
            _isCrouching = false;
            _controller.height = 2f;
        }
        
        // Move controller
        _controller.Move(moveDirection * _currentSpeed * Time.deltaTime);
        
        // Jump
        if (_jumpPressed && _isGrounded && !_isCrouching)
        {
            _velocity.y = Mathf.Sqrt(_jumpHeight * -2f * _gravity);
            _audioSource.PlayOneShot(_jumpSound);
            _isJumping = true;
        }
        
        // Apply gravity
        _velocity.y += _gravity * Time.deltaTime;
        _controller.Move(_velocity * Time.deltaTime);
        
        // Running (uses energy)
        if (_runPressed && _moveInput.magnitude > 0.1f && _currentEnergy > 0)
        {
            _isRunning = true;
            _currentEnergy -= Time.deltaTime * 15f;
        }
        else
        {
            _isRunning = false;
        }
    }
    
    private void HandleRotation()
    {
        // Horizontal rotation (player body)
        float mouseX = _lookInput.x * InputManager.Instance.GetSensitivity() * Time.deltaTime;
        transform.Rotate(Vector3.up * mouseX);
        
        // Vertical rotation (camera)
        if (_cameraTransform != null)
        {
            float mouseY = _lookInput.y * InputManager.Instance.GetSensitivity() * Time.deltaTime;
            
            // Get current rotation
            Vector3 rotation = _cameraTransform.localEulerAngles;
            rotation.x -= mouseY;
            
            // Clamp vertical rotation
            rotation.x = Mathf.Clamp(rotation.x > 180 ? rotation.x - 360 : rotation.x, -80f, 80f);
            
            _cameraTransform.localEulerAngles = rotation;
        }
    }
    
    private void HandleMechaAbilities()
    {
        // Energy regeneration
        if (!_isRunning && _currentEnergy < _energyCapacity)
        {
            _currentEnergy += _energyRegenRate * Time.deltaTime;
            _currentEnergy = Mathf.Min(_currentEnergy, _energyCapacity);
        }
        
        // Mecha-specific abilities
        switch (_mechaType)
        {
            case MechaType.Light:
                // Dash ability
                if (InputManager.Instance.GetButtonDown("Ability1") && _currentEnergy > 30)
                {
                    Dash();
                    _currentEnergy -= 30;
                }
                break;
                
            case MechaType.Medium:
                // Shield ability
                if (InputManager.Instance.GetButtonDown("Ability1") && _currentEnergy > 40)
                {
                    ActivateShield();
                    _currentEnergy -= 40;
                }
                break;
                
            case MechaType.Heavy:
                // Overcharge ability
                if (InputManager.Instance.GetButtonDown("Ability1") && _currentEnergy > 50)
                {
                    Overcharge();
                    _currentEnergy -= 50;
                }
                break;
        }
    }
    
    private void Dash()
    {
        Vector3 dashDirection = transform.forward * _moveInput.y + transform.right * _moveInput.x;
        if (dashDirection.magnitude < 0.1f) dashDirection = transform.forward;
        
        _velocity += dashDirection.normalized * 15f;
        StartCoroutine(DashEffect());
    }
    
    private IEnumerator DashEffect()
    {
        // Visual and sound effects for dash
        GetComponent<TrailRenderer>().emitting = true;
        yield return new WaitForSeconds(0.3f);
        GetComponent<TrailRenderer>().emitting = false;
    }
    
    private void ActivateShield()
    {
        _currentArmor += 50f;
        StartCoroutine(ShieldEffect());
    }
    
    private IEnumerator ShieldEffect()
    {
        GameObject shield = Instantiate(Resources.Load<GameObject>("ShieldEffect"), transform);
        shield.transform.localScale = Vector3.one * 2f;
        
        yield return new WaitForSeconds(5f);
        
        _currentArmor -= 50f;
        Destroy(shield);
    }
    
    private void Overcharge()
    {
        // Increase weapon damage temporarily
        if (_weaponSystem != null)
        {
            _weaponSystem.Overcharge(5f);
        }
        
        StartCoroutine(OverchargeEffect());
    }
    
    private IEnumerator OverchargeEffect()
    {
        ParticleSystem overchargeParticles = GetComponentInChildren<ParticleSystem>();
        if (overchargeParticles != null)
        {
            overchargeParticles.Play();
            yield return new WaitForSeconds(5f);
            overchargeParticles.Stop();
        }
    }
    
    public void TakeDamage(float damage, int attackerID)
    {
        if (_isDead) return;
        
        // Apply armor reduction
        float armorDamage = Mathf.Min(damage * 0.7f, _currentArmor);
        float healthDamage = damage - armorDamage;
        
        _currentArmor -= armorDamage;
        _currentHealth -= healthDamage;
        
        // Show damage indicator
        UIManager.Instance.ShowDamageIndicator(transform.position);
        
        // Play hit sound
        _audioSource.PlayOneShot(Resources.Load<AudioClip>("HitSound"));
        
        // Check death
        if (_currentHealth <= 0)
        {
            Die(attackerID);
        }
    }
    
    private void Die(int killerID)
    {
        _isDead = true;
        
        // Disable controller
        _controller.enabled = false;
        
        // Death effect
        if (_deathEffect != null)
        {
            Instantiate(_deathEffect, transform.position, Quaternion.identity);
        }
        
        // Play death sound
        _audioSource.PlayOneShot(Resources.Load<AudioClip>("DeathSound"));
        
        // Notify game manager
        GameManager.Instance.OnPlayerDeath(_playerData.PlayerID, killerID);
        
        // Disable game object after delay
        StartCoroutine(DisableAfterDelay(5f));
    }
    
    private IEnumerator DisableAfterDelay(float delay)
    {
        yield return new WaitForSeconds(delay);
        gameObject.SetActive(false);
    }
    
    public void Respawn()
    {
        _isDead = false;
        _controller.enabled = true;
        _currentHealth = _mechaHealth;
        _currentArmor = _mechaArmor;
        _currentEnergy = _energyCapacity;
        gameObject.SetActive(true);
    }
    
    private void SetMechaStats(MechaType type)
    {
        switch (type)
        {
            case MechaType.Light:
                _walkSpeed = 7f;
                _runSpeed = 12f;
                _mechaHealth = 80f;
                _mechaArmor = 30f;
                _energyCapacity = 120f;
                _energyRegenRate = 15f;
                break;
                
            case MechaType.Medium:
                _walkSpeed = 5f;
                _runSpeed = 8f;
                _mechaHealth = 100f;
                _mechaArmor = 50f;
                _energyCapacity = 100f;
                _energyRegenRate = 10f;
                break;
                
            case MechaType.Heavy:
                _walkSpeed = 3f;
                _runSpeed = 5f;
                _mechaHealth = 150f;
                _mechaArmor = 100f;
                _energyCapacity = 80f;
                _energyRegenRate = 8f;
                break;
        }
    }
    
    private void SetTeamColor(Team team)
    {
        Renderer[] renderers = GetComponentsInChildren<Renderer>();
        Color teamColor = team == Team.Red ? Color.red : Color.blue;
        
        foreach (Renderer renderer in renderers)
        {
            foreach (Material material in renderer.materials)
            {
                if (material.HasProperty("_Color"))
                {
                    material.color = teamColor;
                }
            }
        }
    }
    
    private void UpdateUI()
    {
        UIManager.Instance.UpdateHealth(_currentHealth, _mechaHealth);
        UIManager.Instance.UpdateArmor(_currentArmor, _mechaArmor);
        UIManager.Instance.UpdateEnergy(_currentEnergy, _energyCapacity);
    }
    
    public Transform GetWeaponHolder() => _weaponHolder;
    public bool IsAlive() => !_isDead;
    public int GetPlayerID() => _playerData?.PlayerID ?? -1;
}

public enum MechaType { Light, Medium, Heavy }// WeaponSystem.cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class WeaponSystem : MonoBehaviour
{
    [System.Serializable]
    public class WeaponData
    {
        public string WeaponName;
        public WeaponType Type;
        public float Damage;
        public float FireRate; // Rounds per second
        public int MagazineSize;
        public int MaxAmmo;
        public float ReloadTime;
        public float Accuracy;
        public float Recoil;
        public float Range;
        public GameObject ModelPrefab;
        public AudioClip FireSound;
        public AudioClip ReloadSound;
        public ParticleSystem MuzzleFlash;
        public GameObject BulletTrail;
    }
    
    [Header("Weapons")]
    [SerializeField] private List<WeaponData> _weaponList = new();
    [SerializeField] private Transform _bulletSpawnPoint;
    
    [Header("Settings")]
    [SerializeField] private float _recoilRecoverySpeed = 5f;
    [SerializeField] private float _aimFOV = 45f;
    [SerializeField] private float _normalFOV = 60f;
    
    // References
    private PlayerController _playerController;
    private Camera _playerCamera;
    private AudioSource _audioSource;
    
    // Current weapon state
    private WeaponData _currentWeapon;
    private GameObject _currentWeaponModel;
    private int _currentAmmo;
    private int _reserveAmmo;
    private bool _isReloading;
    private bool _isAiming;
    private bool _isFiring;
    private float _fireTimer;
    private float _reloadTimer;
    private float _currentRecoil;
    private float _currentSpread;
    
    // Weapon slots
    private Dictionary<int, WeaponData> _weaponSlots = new();
    private int _currentSlot = 0;
    
    // Overcharge
    private bool _isOvercharged;
    private float _overchargeTimer;
    
    public void Initialize(PlayerController playerController)
    {
        _playerController = playerController;
        _playerCamera = Camera.main;
        _audioSource = GetComponent<AudioSource>();
        
        // Setup default weapons
        SetupDefaultWeapons();
        
        // Equip first weapon
        EquipWeapon(0);
    }
    
    private void SetupDefaultWeapons()
    {
        // AK-47
        _weaponSlots[0] = new WeaponData
        {
            WeaponName = "AK-47",
            Type = WeaponType.AssaultRifle,
            Damage = 25f,
            FireRate = 10f,
            MagazineSize = 30,
            MaxAmmo = 180,
            ReloadTime = 2.5f,
            Accuracy = 0.8f,
            Recoil = 1.5f,
            Range = 100f,
            ModelPrefab = Resources.Load<GameObject>("Weapons/AK47"),
            FireSound = Resources.Load<AudioClip>("Sounds/AK47_Fire"),
            ReloadSound = Resources.Load<AudioClip>("Sounds/Reload")
        };
        
        // AWM Sniper
        _weaponSlots[1] = new WeaponData
        {
            WeaponName = "AWM",
            Type = WeaponType.SniperRifle,
            Damage = 100f,
            FireRate = 1f,
            MagazineSize = 5,
            MaxAmmo = 25,
            ReloadTime = 3.8f,
            Accuracy = 0.95f,
            Recoil = 3f,
            Range = 300f,
            ModelPrefab = Resources.Load<GameObject>("Weapons/AWM"),
            FireSound = Resources.Load<AudioClip>("Sounds/AWM_Fire"),
            ReloadSound = Resources.Load<AudioClip>("Sounds/Reload_Sniper")
        };
        
        // MK49 Heavy
        _weaponSlots[2] = new WeaponData
        {
            WeaponName = "MK49",
            Type = WeaponType.HeavyMachineGun,
            Damage = 22f,
            FireRate = 13.3f,
            MagazineSize = 100,
            MaxAmmo = 300,
            ReloadTime = 5f,
            Accuracy = 0.7f,
            Recoil = 2f,
            Range = 80f,
            ModelPrefab = Resources.Load<GameObject>("Weapons/MK49"),
            FireSound = Resources.Load<AudioClip>("Sounds/MK49_Fire"),
            ReloadSound = Resources.Load<AudioClip>("Sounds/Reload_Heavy")
        };
        
        // M82B Anti-Mecha
        _weaponSlots[3] = new WeaponData
        {
            WeaponName = "M82B",
            Type = WeaponType.AntiMechaRifle,
            Damage = 150f,
            FireRate = 0.66f,
            MagazineSize = 5,
            MaxAmmo = 20,
            ReloadTime = 4.2f,
            Accuracy = 0.9f,
            Recoil = 4f,
            Range = 250f,
            ModelPrefab = Resources.Load<GameObject>("Weapons/M82B"),
            FireSound = Resources.Load<AudioClip>("Sounds/M82B_Fire"),
            ReloadSound = Resources.Load<AudioClip>("Sounds/Reload_AntiMecha")
        };
        
        // Initialize ammo
        foreach (var slot in _weaponSlots)
        {
            slot.Value.MaxAmmo = slot.Value.MagazineSize * 6;
        }
    }
    
    private void Update()
    {
        if (_playerController == null || !_playerController.IsAlive()) return;
        
        UpdateTimers();
        UpdateRecoil();
        UpdateSpread();
        
        // Handle firing
        if (_isFiring && CanFire())
        {
            Fire();
        }
        
        // Handle overcharge
        if (_isOvercharged)
        {
            _overchargeTimer -= Time.deltaTime;
            if (_overchargeTimer <= 0)
            {
                _isOvercharged = false;
            }
        }
        
        // Switch weapons
        if (InputManager.Instance.GetButtonDown("Weapon1")) EquipWeapon(0);
        if (InputManager.Instance.GetButtonDown("Weapon2")) EquipWeapon(1);
        if (InputManager.Instance.GetButtonDown("Weapon3")) EquipWeapon(2);
        if (InputManager.Instance.GetButtonDown("Weapon4")) EquipWeapon(3);
        
        // Update UI
        UpdateWeaponUI();
    }
    
    public void StartFiring()
    {
        if (_currentWeapon.Type == WeaponType.SniperRifle || 
            _currentWeapon.Type == WeaponType.AntiMechaRifle)
        {
            // Single shot for snipers
            if (CanFire() && !_isReloading)
            {
                Fire();
            }
        }
        else
        {
            _isFiring = true;
        }
    }
    
    public void StopFiring()
    {
        _isFiring = false;
    }
    
    public void SetAiming(bool aiming)
    {
        _isAiming = aiming;
        
        if (_playerCamera != null)
        {
            _playerCamera.fieldOfView = aiming ? _aimFOV : _normalFOV;
        }
        
        // Show/hide scope overlay for snipers
        if (_currentWeapon.Type == WeaponType.SniperRifle || 
            _currentWeapon.Type == WeaponType.AntiMechaRifle)
        {
            UIManager.Instance.SetScopeOverlay(aiming);
        }
    }
    
    private void Fire()
    {
        if (_currentWeapon == null || _currentAmmo <= 0 || _isReloading) return;
        
        // Consume ammo
        _currentAmmo--;
        
        // Play fire sound
        if (_audioSource != null && _currentWeapon.FireSound != null)
        {
            _audioSource.PlayOneShot(_currentWeapon.FireSound);
        }
        
        // Muzzle flash
        if (_currentWeapon.MuzzleFlash != null)
        {
            ParticleSystem muzzleFlash = Instantiate(_currentWeapon.MuzzleFlash, _bulletSpawnPoint);
            Destroy(muzzleFlash.gameObject, 0.1f);
        }
        
        // Calculate spread
        Vector3 spread = CalculateSpread();
        Vector3 fireDirection = (_bulletSpawnPoint.forward + spread).normalized;
        
        // Create bullet trail
        if (_currentWeapon.BulletTrail != null)
        {
            GameObject trail = Instantiate(_currentWeapon.BulletTrail, _bulletSpawnPoint.position, Quaternion.identity);
            StartCoroutine(SpawnTrail(trail, fireDirection));
        }
        
        // Raycast for hit detection
        RaycastHit hit;
        if (Physics.Raycast(_bulletSpawnPoint.position, fireDirection, out hit, _currentWeapon.Range))
        {
            // Apply damage
            if (hit.collider.CompareTag("Player"))
            {
                PlayerController target = hit.collider.GetComponent<PlayerController>();
                if (target != null)
                {
                    float damage = _currentWeapon.Damage;
                    
                    // Headshot multiplier
                    if (hit.collider is CapsuleCollider capsule && 
                        hit.point.y > capsule.bounds.center.y + capsule.height * 0.3f)
                    {
                        damage *= 2f;
                    }
                    
                    // Apply overcharge bonus
                    if (_isOvercharged)
                    {
                        damage *= 1.5f;
                    }
                    
                    target.TakeDamage(damage, _playerController.GetPlayerID());
                    
                    // Show hit marker
                    UIManager.Instance.ShowHitMarker();
                }
            }
            
            // Impact effect
            GameObject impact = Instantiate(Resources.Load<GameObject>("BulletImpact"), hit.point, 
                Quaternion.LookRotation(hit.normal));
            Destroy(impact, 2f);
        }
        
        // Apply recoil
        _currentRecoil += _currentWeapon.Recoil;
        
        // Reset fire timer
        _fireTimer = 1f / _currentWeapon.FireRate;
        
        // Check for empty magazine
        if (_currentAmmo <= 0)
        {
            Reload();
        }
    }
    
    private IEnumerator SpawnTrail(GameObject trail, Vector3 direction)
    {
        float distance = _currentWeapon.Range;
        float time = 0;
        Vector3 startPosition = trail.transform.position;
        
        while (time < 1)
        {
            trail.transform.position = Vector3.Lerp(startPosition, startPosition + direction * distance, time);
            time += Time.deltaTime / 0.1f; // Trail duration
            yield return null;
        }
        
        Destroy(trail);
    }
    
    private Vector3 CalculateSpread()
    {
        float baseSpread = 1f - _currentWeapon.Accuracy;
        float movementSpread = _playerController.GetMovementMagnitude() * 0.1f;
        float recoilSpread = _currentRecoil * 0.01f;
        
        float totalSpread = (baseSpread + movementSpread + recoilSpread) * (_isAiming ? 0.3f : 1f);
        
        return new Vector3(
            Random.Range(-totalSpread, totalSpread),
            Random.Range(-totalSpread, totalSpread),
            0
        );
    }
    
    private void UpdateRecoil()
    {
        if (_currentRecoil > 0)
        {
            _currentRecoil = Mathf.Lerp(_currentRecoil, 0, _recoilRecoverySpeed * Time.deltaTime);
        }
    }
    
    private void UpdateSpread()
    {
        // Spread recovery when not firing
        if (!_isFiring && _currentSpread > 0)
        {
            _currentSpread = Mathf.Lerp(_currentSpread, 0, 2f * Time.deltaTime);
        }
    }
    
    private void UpdateTimers()
    {
        if (_fireTimer > 0)
        {
            _fireTimer -= Time.deltaTime;
        }
        
        if (_isReloading)
        {
            _reloadTimer -= Time.deltaTime;
            
            // Update reload UI
            UIManager.Instance.UpdateReloadProgress(1f - (_reloadTimer / _currentWeapon.ReloadTime));
            
            if (_reloadTimer <= 0)
            {
                CompleteReload();
            }
        }
    }
    
    public void Reload()
    {
        if (_isReloading || _currentAmmo == _currentWeapon.MagazineSize || _reserveAmmo <= 0) return;
        
        _isReloading = true;
        _reloadTimer = _currentWeapon.ReloadTime;
        
        // Play reload sound
        if (_audioSource != null && _currentWeapon.ReloadSound != null)
        {
            _audioSource.PlayOneShot(_currentWeapon.ReloadSound);
        }
        
        // Trigger reload animation
        // Animation logic here
    }
    
    private void CompleteReload()
    {
        int ammoNeeded = _currentWeapon.MagazineSize - _currentAmmo;
        int ammoToAdd = Mathf.Min(ammoNeeded, _reserveAmmo);
        
        _currentAmmo += ammoToAdd;
        _reserveAmmo -= ammoToAdd;
        _isReloading = false;
    }
    
    private bool CanFire()
    {
        return _fireTimer <= 0 && !_isReloading && _currentAmmo > 0;
    }
    
    public void EquipWeapon(int slot)
    {
        if (!_weaponSlots.ContainsKey(slot)) return;
        
        // Destroy current weapon model
        if (_currentWeaponModel != null)
        {
            Destroy(_currentWeaponModel);
        }
        
        // Set new weapon
        _currentWeapon = _weaponSlots[slot];
        _currentSlot = slot;
        
        // Reset ammo if new weapon
        if (_currentAmmo == 0)
        {
            _currentAmmo = _currentWeapon.MagazineSize;
            _reserveAmmo = _currentWeapon.MaxAmmo - _currentAmmo;
        }
        
        // Spawn weapon model
        if (_currentWeapon.ModelPrefab != null)
        {
            Transform weaponHolder = _playerController.GetWeaponHolder();
            if (weaponHolder != null)
            {
                _currentWeaponModel = Instantiate(_currentWeapon.ModelPrefab, weaponHolder);
            }
        }
        
        // Reset aim
        SetAiming(false);
        
        // Update UI
        UpdateWeaponUI();
    }
    
    public void Overcharge(float duration)
    {
        _isOvercharged = true;
        _overchargeTimer = duration;
        
        // Visual effect
        StartCoroutine(OverchargeEffect());
    }
    
    private IEnumerator OverchargeEffect()
    {
        // Weapon glow effect
        if (_currentWeaponModel != null)
        {
            Renderer renderer = _currentWeaponModel.GetComponent<Renderer>();
            if (renderer != null)
            {
                Color originalColor = renderer.material.color;
                renderer.material.color = Color.yellow;
                
                yield return new WaitForSeconds(5f);
                
                renderer.material.color = originalColor;
            }
        }
    }
    
    private void UpdateWeaponUI()
    {
        UIManager.Instance.UpdateAmmo(_currentAmmo, _currentWeapon.MagazineSize, _reserveAmmo);
        UIManager.Instance.UpdateWeaponName(_currentWeapon.WeaponName);
        
        // Show/hide crosshair based on weapon type
        bool showCrosshair = _currentWeapon.Type != WeaponType.SniperRifle && 
                            _currentWeapon.Type != WeaponType.AntiMechaRifle;
        UIManager.Instance.SetCrosshairVisible(showCrosshair);
    }
    
    public void AddAmmo(int amount)
    {
        _reserveAmmo = Mathf.Min(_reserveAmmo + amount, _currentWeapon.MaxAmmo);
    }
}

public enum WeaponType 
{ 
    AssaultRifle, 
    SniperRifle, 
    HeavyMachineGun, 
    AntiMechaRifle,
    Pistol,
    Shotgun
}// AIController.cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AI;

public class AIController : MonoBehaviour
{
    [Header("AI Settings")]
    [SerializeField] private AIDifficulty _difficulty = AIDifficulty.Medium;
    [SerializeField] private float _sightRange = 30f;
    [SerializeField] private float _attackRange = 20f;
    [SerializeField] private float _patrolRadius = 50f;
    
    [Header("Behavior")]
    [SerializeField] private float _decisionInterval = 0.5f;
    [SerializeField] private float _targetUpdateInterval = 1f;
    
    // Components
    private NavMeshAgent _navAgent;
    private PlayerController _playerController;
    private WeaponSystem _weaponSystem;
    
    // State
    private AIState _currentState = AIState.Patrol;
    private Transform _currentTarget;
    private Vector3 _patrolCenter;
    private float _stateTimer;
    private float _decisionTimer;
    private float _targetUpdateTimer;
    
    // Memory
    private List<Transform> _knownPlayers = new();
    private Vector3 _lastKnownPosition;
    private float _targetLostTime;
    
    public enum AIState { Patrol, Chase, Attack, Flee, Hide, Dead }
    public enum AIDifficulty { Easy, Medium, Hard, Expert }
    
    private void Start()
    {
        _navAgent = GetComponent<NavMeshAgent>();
        _playerController = GetComponent<PlayerController>();
        _weaponSystem = GetComponent<WeaponSystem>();
        
        // Set difficulty
        SetDifficulty(_difficulty);
        
        // Initialize patrol center
        _patrolCenter = transform.position;
        
        // Start AI behavior
        StartCoroutine(AIBehaviorLoop());
    }
    
    private void SetDifficulty(AIDifficulty difficulty)
    {
        switch (difficulty)
        {
            case AIDifficulty.Easy:
                _sightRange = 20f;
                _attackRange = 15f;
                _decisionInterval = 1f;
                _navAgent.speed = 3f;
                break;
                
            case AIDifficulty.Medium:
                _sightRange = 30f;
                _attackRange = 20f;
                _decisionInterval = 0.5f;
                _navAgent.speed = 4f;
                break;
                
            case AIDifficulty.Hard:
                _sightRange = 40f;
                _attackRange = 25f;
                _decisionInterval = 0.3f;
                _navAgent.speed = 5f;
                break;
                
            case AIDifficulty.Expert:
                _sightRange = 50f;
                _attackRange = 30f;
                _decisionInterval = 0.2f;
                _navAgent.speed = 6f;
                break;
        }
    }
    
    private IEnumerator AIBehaviorLoop()
    {
        while (_playerController.IsAlive())
        {
            UpdatePerception();
            UpdateState();
            ExecuteState();
            
            yield return new WaitForSeconds(_decisionInterval);
        }
        
        // AI is dead
        _currentState = AIState.Dead;
        _navAgent.isStopped = true;
    }
    
    private void UpdatePerception()
    {
        // Find players in sight range
        Collider[] colliders = Physics.OverlapSphere(transform.position, _sightRange);
        _knownPlayers.Clear();
        
        foreach (var collider in colliders)
        {
            if (collider.CompareTag("Player") && collider.gameObject != gameObject)
            {
                // Check line of sight
                Vector3 direction = collider.transform.position - transform.position;
                if (!Physics.Raycast(transform.position, direction.normalized, direction.magnitude, 
                    LayerMask.GetMask("Obstacle")))
                {
                    _knownPlayers.Add(collider.transform);
                    
                    // Update target if closer or more dangerous
                    if (_currentTarget == null || 
                        Vector3.Distance(transform.position, collider.transform.position) < 
                        Vector3.Distance(transform.position, _currentTarget.position))
                    {
                        _currentTarget = collider.transform;
                        _lastKnownPosition = _currentTarget.position;
                        _targetLostTime = 0f;
                    }
                }
            }
        }
        
        // Track target lost time
        if (_currentTarget != null && !_knownPlayers.Contains(_currentTarget))
        {
            _targetLostTime += _decisionInterval;
            
            if (_targetLostTime > 5f) // Forget target after 5 seconds
            {
                _currentTarget = null;
            }
        }
    }
    
    private void UpdateState()
    {
        float healthPercent = _playerController.GetHealthPercent();
        
        switch (_currentState)
        {
            case AIState.Patrol:
                if (_currentTarget != null)
                {
                    _currentState = AIState.Chase;
                }
                else if (healthPercent < 0.3f && Random.value < 0.3f)
                {
                    _currentState = AIState.Hide;
                }
                break;
                
            case AIState.Chase:
                if (_currentTarget == null)
                {
                    _currentState = AIState.Patrol;
                }
                else if (Vector3.Distance(transform.position, _currentTarget.position) <= _attackRange)
                {
                    _currentState = AIState.Attack;
                }
                else if (healthPercent < 0.2f)
                {
                    _currentState = AIState.Flee;
                }
                break;
                
            case AIState.Attack:
                if (_currentTarget == null || 
                    Vector3.Distance(transform.position, _currentTarget.position) > _attackRange * 1.2f)
                {
                    _currentState = AIState.Chase;
                }
                else if (healthPercent < 0.3f && _knownPlayers.Count > 1)
                {
                    _currentState = AIState.Flee;
                }
                break;
                
            case AIState.Flee:
                if (healthPercent > 0.5f || _knownPlayers.Count == 0)
                {
                    _currentState = AIState.Patrol;
                }
                break;
                
            case AIState.Hide:
                if (healthPercent > 0.7f || _stateTimer > 10f)
                {
                    _currentState = AIState.Patrol;
                }
                break;
        }
        
        _stateTimer += _decisionInterval;
    }
    
    private void ExecuteState()
    {
        switch (_currentState)
        {
            case AIState.Patrol:
                Patrol();
                break;
                
            case AIState.Chase:
                Chase();
                break;
                
            case AIState.Attack:
                Attack();
                break;
                
            case AIState.Flee:
                Flee();
                break;
                
            case AIState.Hide:
                Hide();
                break;
        }
    }
    
    private void Patrol()
    {
        if (!_navAgent.hasPath || _navAgent.remainingDistance < 2f)
        {
            Vector3 randomPoint = _patrolCenter + Random.insideUnitSphere * _patrolRadius;
            randomPoint.y = 0;
            
            NavMeshHit hit;
            if (NavMesh.SamplePosition(randomPoint, out hit, 10f, NavMesh.AllAreas))
            {
                _navAgent.SetDestination(hit.position);
            }
        }
    }
    
    private void Chase()
    {
        if (_currentTarget != null)
        {
            _navAgent.SetDestination(_currentTarget.position);
            
            // Look at target
            Vector3 direction = (_currentTarget.position - transform.position).normalized;
            transform.rotation = Quaternion.Lerp(transform.rotation, 
                Quaternion.LookRotation(new Vector3(direction.x, 0, direction.z)), 
                5f * Time.deltaTime);
        }
        else if (_lastKnownPosition != Vector3.zero)
        {
            _navAgent.SetDestination(_lastKnownPosition);
            
            if (Vector3.Distance(transform.position, _lastKnownPosition) < 2f)
            {
                _lastKnownPosition = Vector3.zero;
            }
        }
    }
    
    private void Attack()
    {
        if (_currentTarget != null)
        {
            // Stop moving when in range
            _navAgent.isStopped = true;
            
            // Look at target
            Vector3 direction = (_currentTarget.position - transform.position).normalized;
            transform.rotation = Quaternion.Lerp(transform.rotation, 
                Quaternion.LookRotation(new Vector3(direction.x, 0, direction.z)), 
                10f * Time.deltaTime);
            
            // Fire weapon
            if (_weaponSystem != null && Random.value < 0.8f)
            {
                _weaponSystem.StartFiring();
            }
            
            // Strafe movement
            if (Random.value < 0.1f)
            {
                Vector3 strafeDirection = transform.right * (Random.value > 0.5f ? 1 : -1);
                transform.position += strafeDirection * 2f * Time.deltaTime;
            }
        }
        else
        {
            _navAgent.isStopped = false;
        }
    }
    
    private void Flee()
    {
        if (_knownPlayers.Count > 0)
        {
            // Find direction away from all players
            Vector3 fleeDirection = Vector3.zero;
            
            foreach (var player in _knownPlayers)
            {
                fleeDirection += (transform.position - player.position).normalized;
            }
            
            fleeDirection.Normalize();
            
            // Find flee position
            Vector3 fleePosition = transform.position + fleeDirection * 20f;
            
            NavMeshHit hit;
            if (NavMesh.SamplePosition(fleePosition, out hit, 10f, NavMesh.AllAreas))
            {
                _navAgent.SetDestination(hit.position);
            }
        }
    }
    
    private void Hide()
    {
        // Find cover position
        Collider[] coverSpots = Physics.OverlapSphere(transform.position, 20f, LayerMask.GetMask("Cover"));
        
        if (coverSpots.Length > 0)
        {
            Transform bestCover = coverSpots[0].transform;
            
            _navAgent.SetDestination(bestCover.position);
            
            // Stop near cover
            if (Vector3.Distance(transform.position, bestCover.position) < 2f)
            {
                _navAgent.isStopped = true;
                
                // Regenerate health
                _playerController.Heal(1f * _decisionInterval);
            }
        }
        else
        {
            Patrol();
        }
    }
    
    public void OnDamageTaken(Transform attacker)
    {
        // React to being shot
        _currentTarget = attacker;
        _currentState = AIState.Chase;
        
        // Alert nearby AI
        AlertNearbyAI(attacker);
    }
    
    private void AlertNearbyAI(Transform threat)
    {
        Collider[] nearbyAI = Physics.OverlapSphere(transform.position, 30f);
        
        foreach (var collider in nearbyAI)
        {
            AIController ai = collider.GetComponent<AIController>();
            if (ai != null && ai != this)
            {
                ai._currentTarget = threat;
                ai._currentState = AIState.Chase;
            }
        }
    }
    
    private void OnDrawGizmosSelected()
    {
        // Draw sight range
        Gizmos.color = Color.yellow;
        Gizmos.DrawWireSphere(transform.position, _sightRange);
        
        // Draw attack range
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(transform.position, _attackRange);
        
        // Draw current target line
        if (_currentTarget != null)
        {
            Gizmos.color = Color.green;
            Gizmos.DrawLine(transform.position, _currentTarget.position);
        }
    }
}// NetworkManager.cs
using Fusion;
using System.Collections.Generic;
using UnityEngine;

public class NetworkManager : MonoBehaviour
{
    public static NetworkManager Instance { get; private set; }
    
    [Header("Network Settings")]
    [SerializeField] private NetworkRunner _runnerPrefab;
    [SerializeField] private NetworkObject _playerPrefab;
    [SerializeField] private int _maxPlayers = 15;
    
    [Header("Room Settings")]
    [SerializeField] private string _defaultRoomName = "MechaSphere_Room";
    [SerializeField] private bool _isPrivateRoom = false;
    
    // Network runner instance
    private NetworkRunner _runner;
    
    // Player tracking
    private Dictionary<PlayerRef, NetworkObject> _spawnedPlayers = new();
    
    // Game state
    private GameMode _currentGameMode;
    private string _currentMap;
    private bool _isHost;
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    public async void StartHost(GameMode gameMode, string mapName, string roomName = "")
    {
        if (_runner != null)
        {
            Debug.LogWarning("Network runner already exists!");
            return;
        }
        
        _runner = Instantiate(_runnerPrefab);
        _runner.name = "Network Runner";
        DontDestroyOnLoad(_runner);
        
        _isHost = true;
        _currentGameMode = gameMode;
        _currentMap = mapName;
        
        // Start game as host
        var startGameArgs = new StartGameArgs()
        {
            GameMode = GameMode.Host,
            SessionName = string.IsNullOrEmpty(roomName) ? _defaultRoomName : roomName,
            PlayerCount = _maxPlayers,
            SceneManager = _runner.gameObject.AddComponent<NetworkSceneManagerDefault>(),
            Scene = SceneRef.FromIndex(SceneManager.GetSceneByName(mapName).buildIndex)
        };
        
        var result = await _runner.StartGame(startGameArgs);
        
        if (result.Ok)
        {
            Debug.Log($"Host started successfully: {startGameArgs.SessionName}");
            
            // Spawn host player
            SpawnPlayer(_runner.LocalPlayer);
            
            // Load game scene
            _runner.SetActiveScene(mapName);
        }
        else
        {
            Debug.LogError($"Failed to start host: {result.ShutdownReason}");
        }
    }
    
    public async void JoinGame(string roomName)
    {
        if (_runner != null)
        {
            Debug.LogWarning("Network runner already exists!");
            return;
        }
        
        _runner = Instantiate(_runnerPrefab);
        _runner.name = "Network Runner";
        DontDestroyOnLoad(_runner);
        
        _isHost = false;
        
        // Join existing game
        var startGameArgs = new StartGameArgs()
        {
            GameMode = GameMode.Client,
            SessionName = roomName
        };
        
        var result = await _runner.StartGame(startGameArgs);
        
        if (result.Ok)
        {
            Debug.Log($"Joined game successfully: {roomName}");
        }
        else
        {
            Debug.LogError($"Failed to join game: {result.ShutdownReason}");
        }
    }
    
    public void CreateRoom(string roomName, GameMode gameMode, string mapName, int maxPlayers, bool isPrivate)
    {
        // For cloud-based matchmaking
        StartCoroutine(CreateRoomCoroutine(roomName, gameMode, mapName, maxPlayers, isPrivate));
    }
    
    private System.Collections.IEnumerator CreateRoomCoroutine(string roomName, GameMode gameMode, string mapName, int maxPlayers, bool isPrivate)
    {
        // Implement cloud room creation
        yield return null;
    }
    
    public void FindMatch(GameMode gameMode)
    {
        // Quick matchmaking
        StartCoroutine(FindMatchCoroutine(gameMode));
    }
    
    private System.Collections.IEnumerator FindMatchCoroutine(GameMode gameMode)
    {
        // Implement matchmaking logic
        yield return null;
    }
    
    public void SpawnPlayer(PlayerRef player)
    {
        if (!_runner.IsServer) return;
        
        // Get spawn position
        Vector3 spawnPosition = GetSpawnPosition(player);
        
        // Spawn player object
        NetworkObject playerObject = _runner.Spawn(_playerPrefab, spawnPosition, Quaternion.identity, player);
        _spawnedPlayers.Add(player, playerObject);
        
        Debug.Log($"Player spawned: {player.PlayerId}");
    }
    
    public void DespawnPlayer(PlayerRef player)
    {
        if (_spawnedPlayers.TryGetValue(player, out NetworkObject playerObject))
        {
            _runner.Despawn(playerObject);
            _spawnedPlayers.Remove(player);
            
            Debug.Log($"Player despawned: {player.PlayerId}");
        }
    }
    
    private Vector3 GetSpawnPosition(PlayerRef player)
    {
        // Team-based or random spawn logic
        if (_currentGameMode == GameMode.TeamDeathmatch)
        {
            Team team = GetPlayerTeam(player);
            Transform[] teamSpawns = GameObject.FindGameObjectsWithTag(team == Team.Red ? "Spawn_Red" : "Spawn_Blue")
                .TransformArray(t => t.transform);
            
            if (teamSpawns.Length > 0)
            {
                return teamSpawns[Random.Range(0, teamSpawns.Length)].position;
            }
        }
        
        // Random spawn for FFA
        GameObject[] spawnPoints = GameObject.FindGameObjectsWithTag("SpawnPoint");
        if (spawnPoints.Length > 0)
        {
            return spawnPoints[Random.Range(0, spawnPoints.Length)].transform.position;
        }
        
        return Vector3.zero;
    }
    
    private Team GetPlayerTeam(PlayerRef player)
    {
        // Simple team assignment - first half red, second half blue
        int playerCount = _runner.ActivePlayers.Count;
        int playerIndex = 0;
        
        foreach (var p in _runner.ActivePlayers)
        {
            if (p == player) break;
            playerIndex++;
        }
        
        return playerIndex < playerCount / 2 ? Team.Red : Team.Blue;
    }
    
    public void LeaveGame()
    {
        if (_runner != null)
        {
            _runner.Shutdown();
            Destroy(_runner.gameObject);
            _runner = null;
        }
        
        // Return to main menu
        SceneManager.LoadScene("MainMenu");
    }
    
    public void SendChatMessage(string message)
    {
        RPC_SendChatMessage(_runner.LocalPlayer, message);
    }
    
    [Rpc(RpcSources.All, RpcTargets.All)]
    private void RPC_SendChatMessage(PlayerRef player, string message, RpcInfo info = default)
    {
        // Display chat message
        UIManager.Instance.AddChatMessage(player.PlayerId, message);
    }
    
    public void SendKillFeed(string killer, string victim, string weapon)
    {
        RPC_SendKillFeed(killer, victim, weapon);
    }
    
    [Rpc(RpcSources.All, RpcTargets.All)]
    private void RPC_SendKillFeed(string killer, string victim, string weapon, RpcInfo info = default)
    {
        // Update kill feed
        UIManager.Instance.AddKillFeed(killer, victim, weapon);
    }
    
    public bool IsConnected()
    {
        return _runner != null && _runner.IsConnected;
    }
    
    public bool IsHost()
    {
        return _isHost;
    }
    
    public int GetPlayerCount()
    {
        return _runner?.ActivePlayers.Count ?? 0;
    }
    
    public List<string> GetPlayerNames()
    {
        List<string> names = new List<string>();
        
        if (_runner != null)
        {
            foreach (var player in _runner.ActivePlayers)
            {
                names.Add($"Player_{player.PlayerId}");
            }
        }
        
        return names;
    }
}

// Network Player Script
public class NetworkPlayer : NetworkBehaviour
{
    [Networked] public string PlayerName { get; set; }
    [Networked] public int PlayerID { get; set; }
    [Networked] public Team PlayerTeam { get; set; }
    [Networked] public int Kills { get; set; }
    [Networked] public int Deaths { get; set; }
    [Networked] public int Score { get; set; }
    [Networked] public bool IsAlive { get; set; }
    
    private PlayerController _playerController;
    
    public override void Spawned()
    {
        if (Object.HasInputAuthority)
        {
            // Initialize local player
            _playerController = GetComponent<PlayerController>();
            
            // Set player data
            PlayerData data = new PlayerData
            {
                PlayerID = Object.InputAuthority.PlayerId,
                PlayerObject = gameObject,
                Team = PlayerTeam,
                IsAlive = true
            };
            
            _playerController.Initialize(data);
            
            // Set camera follow
            Camera.main.GetComponent<CameraFollow>().SetTarget(transform);
        }
    }
    
    public override void FixedUpdateNetwork()
    {
        if (GetInput(out NetworkInputData input))
        {
            // Process input for movement and actions
            if (_playerController != null)
            {
                // Update player controller with network input
            }
        }
    }
    
    [Rpc(RpcSources.All, RpcTargets.StateAuthority)]
    public void RPC_TakeDamage(float damage, PlayerRef attacker)
    {
        // Server-side damage calculation
        if (_playerController != null && IsAlive)
        {
            _playerController.TakeDamage(damage, attacker.PlayerId);
            
            if (!_playerController.IsAlive())
            {
                IsAlive = false;
                Deaths++;
                
                // Notify killer
                if (attacker != Object.InputAuthority)
                {
                    NetworkPlayer killer = FindPlayer(attacker);
                    if (killer != null)
                    {
                        killer.Kills++;
                        killer.Score += 100;
                    }
                }
                
                // Broadcast death event
                RPC_OnPlayerDeath(Object.InputAuthority.PlayerId, attacker.PlayerId);
            }
        }
    }
    
    [Rpc(RpcSources.StateAuthority, RpcTargets.All)]
    private void RPC_OnPlayerDeath(int victimID, int killerID)
    {
        // Update UI and game state
        GameManager.Instance.OnPlayerDeath(victimID, killerID);
    }
    
    private NetworkPlayer FindPlayer(PlayerRef playerRef)
    {
        // Find network player by PlayerRef
        foreach (var player in FindObjectsOfType<NetworkPlayer>())
        {
            if (player.Object.InputAuthority == playerRef)
            {
                return player;
            }
        }
        return null;
    }
}

public struct NetworkInputData : INetworkInput
{
    public Vector2 MoveDirection;
    public Vector2 LookDirection;
    public NetworkButtons Buttons;
}// SaveManager.cs
using System;
using System.Collections.Generic;
using System.IO;
using UnityEngine;

public class SaveManager : MonoBehaviour
{
    public static SaveManager Instance { get; private set; }
    
    private PlayerData _playerData;
    private string _savePath;
    
    [Serializable]
    public class PlayerData
    {
        public string PlayerName = "Mecha Pilot";
        public int PlayerLevel = 1;
        public int Experience = 0;
        public int TotalXP = 0;
        public int Credits = 1000;
        public int MechaParts = 0;
        
        public Dictionary<string, WeaponData> Weapons = new();
        public Dictionary<string, MechaData> Mechas = new();
        public Dictionary<string, CosmeticData> Cosmetics = new();
        
        public List<string> UnlockedAchievements = new();
        public Dictionary<string, int> WeaponStats = new();
        public Dictionary<string, int> GameStats = new();
        
        public SettingsData Settings = new();
    }
    
    [Serializable]
    public class WeaponData
    {
        public string WeaponID;
        public bool IsUnlocked;
        public int Level;
        public int Kills;
        public int XP;
        public List<string> Attachments = new();
    }
    
    [Serializable]
    public class MechaData
    {
        public string MechaID;
        public bool IsUnlocked;
        public int Level;
        public int XP;
        public float HealthUpgrade;
        public float ArmorUpgrade;
        public float SpeedUpgrade;
        public float EnergyUpgrade;
        public string CurrentSkin;
    }
    
    [Serializable]
    public class CosmeticData
    {
        public string CosmeticID;
        public bool IsUnlocked;
        public bool IsEquipped;
    }
    
    [Serializable]
    public class SettingsData
    {
        public float MasterVolume = 1f;
        public float MusicVolume = 0.8f;
        public float SFXVolume = 0.8f;
        public float Sensitivity = 1f;
        public bool InvertY = false;
        public int GraphicsQuality = 2;
        public string Language = "English";
        public Dictionary<string, KeyCode> Keybindings = new();
    }
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
            
            _savePath = Path.Combine(Application.persistentDataPath, "save_data.json");
            LoadGame();
            
            InitializeDefaultData();
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    private void InitializeDefaultData()
    {
        // Initialize if new player
        if (_playerData == null)
        {
            _playerData = new PlayerData();
            
            // Unlock default mecha
            UnlockMecha("mecha_light");
            UnlockMecha("mecha_medium");
            
            // Unlock default weapons
            UnlockWeapon("weapon_ak47");
            UnlockWeapon("weapon_pistol");
            
            // Initialize game stats
            InitializeStats();
            
            SaveGame();
        }
    }
    
    private void InitializeStats()
    {
        _playerData.GameStats["total_kills"] = 0;
        _playerData.GameStats["total_deaths"] = 0;
        _playerData.GameStats["total_wins"] = 0;
        _playerData.GameStats["total_matches"] = 0;
        _playerData.GameStats["total_time"] = 0;
        _playerData.GameStats["headshots"] = 0;
        _playerData.GameStats["longest_kill_streak"] = 0;
    }
    
    public void SaveGame()
    {
        try
        {
            string json = JsonUtility.ToJson(_playerData, true);
            File.WriteAllText(_savePath, json);
            
            Debug.Log("Game saved successfully");
        }
        catch (Exception e)
        {
            Debug.LogError($"Failed to save game: {e.Message}");
        }
    }
    
    public void LoadGame()
    {
        try
        {
            if (File.Exists(_savePath))
            {
                string json = File.ReadAllText(_savePath);
                _playerData = JsonUtility.FromJson<PlayerData>(json);
                
                Debug.Log("Game loaded successfully");
            }
            else
            {
                Debug.Log("No save file found, creating new game");
                _playerData = new PlayerData();
            }
        }
        catch (Exception e)
        {
            Debug.LogError($"Failed to load game: {e.Message}");
            _playerData = new PlayerData();
        }
    }
    
    public void AddExperience(int xp)
    {
        _playerData.Experience += xp;
        _playerData.TotalXP += xp;
        
        // Check level up
        int xpForNextLevel = GetXPForNextLevel();
        
        while (_playerData.Experience >= xpForNextLevel)
        {
            _playerData.Experience -= xpForNextLevel;
            _playerData.PlayerLevel++;
            
            // Level up rewards
            OnLevelUp(_playerData.PlayerLevel);
            
            xpForNextLevel = GetXPForNextLevel();
        }
        
        SaveGame();
    }
    
    private int GetXPForNextLevel()
    {
        // Exponential XP requirement
        return 1000 + (_playerData.PlayerLevel * 500);
    }
    
    private void OnLevelUp(int level)
    {
        // Level up rewards
        _playerData.Credits += level * 100;
        _playerData.MechaParts += level * 10;
        
        // Unlock rewards based on level
        switch (level)
        {
            case 5:
                UnlockWeapon("weapon_awm");
                break;
            case 10:
                UnlockMecha("mecha_heavy");
                break;
            case 15:
                UnlockWeapon("weapon_mk49");
                break;
            case 20:
                UnlockWeapon("weapon_m82b");
                break;
        }
        
        // Show level up notification
        UIManager.Instance.ShowNotification($"Level Up! Reached Level {level}");
    }
    
    public void AddCredits(int amount)
    {
        _playerData.Credits += amount;
        SaveGame();
    }
    
    public void AddMechaParts(int amount)
    {
        _playerData.MechaParts += amount;
        SaveGame();
    }
    
    public bool SpendCredits(int amount)
    {
        if (_playerData.Credits >= amount)
        {
            _playerData.Credits -= amount;
            SaveGame();
            return true;
        }
        return false;
    }
    
    public bool SpendMechaParts(int amount)
    {
        if (_playerData.MechaParts >= amount)
        {
            _playerData.MechaParts -= amount;
            SaveGame();
            return true;
        }
        return false;
    }
    
    public void UnlockWeapon(string weaponID)
    {
        if (!_playerData.Weapons.ContainsKey(weaponID))
        {
            _playerData.Weapons[weaponID] = new WeaponData
            {
                WeaponID = weaponID,
                IsUnlocked = true,
                Level = 1,
                Kills = 0,
                XP = 0
            };
        }
        else
        {
            _playerData.Weapons[weaponID].IsUnlocked = true;
        }
        
        SaveGame();
    }
    
    public void UnlockMecha(string mechaID)
    {
        if (!_playerData.Mechas.ContainsKey(mechaID))
        {
            _playerData.Mechas[mechaID] = new MechaData
            {
                MechaID = mechaID,
                IsUnlocked = true,
                Level = 1,
                HealthUpgrade = 0,
                ArmorUpgrade = 0,
                SpeedUpgrade = 0,
                EnergyUpgrade = 0
            };
        }
        else
        {
            _playerData.Mechas[mechaID].IsUnlocked = true;
        }
        
        SaveGame();
    }
    
    public void UnlockCosmetic(string cosmeticID)
    {
        if (!_playerData.Cosmetics.ContainsKey(cosmeticID))
        {
            _playerData.Cosmetics[cosmeticID] = new CosmeticData
            {
                CosmeticID = cosmeticID,
                IsUnlocked = true,
                IsEquipped = false
            };
        }
        else
        {
            _playerData.Cosmetics[cosmeticID].IsUnlocked = true;
        }
        
        SaveGame();
    }
    
    public void AddWeaponKill(string weaponID)
    {
        if (_playerData.WeaponStats.ContainsKey(weaponID))
        {
            _playerData.WeaponStats[weaponID]++;
        }
        else
        {
            _playerData.WeaponStats[weaponID] = 1;
        }
        
        if (_playerData.Weapons.ContainsKey(weaponID))
        {
            _playerData.Weapons[weaponID].Kills++;
            _playerData.Weapons[weaponID].XP += 10;
            
            // Check for weapon level up
            CheckWeaponLevelUp(weaponID);
        }
        
        // Update total kills
        _playerData.GameStats["total_kills"]++;
        
        SaveGame();
    }
    
    private void CheckWeaponLevelUp(string weaponID)
    {
        WeaponData weapon = _playerData.Weapons[weaponID];
        int xpForNextLevel = weapon.Level * 100;
        
        if (weapon.XP >= xpForNextLevel)
        {
            weapon.XP -= xpForNextLevel;
            weapon.Level++;
            
            // Weapon level up rewards
            AddCredits(weapon.Level * 50);
            AddMechaParts(weapon.Level * 5);
        }
    }
    
    public void AddMatchResult(bool won, int kills, int deaths, int score, float matchTime)
    {
        _playerData.GameStats["total_matches"]++;
        _playerData.GameStats["total_time"] += (int)matchTime;
        
        if (won)
        {
            _playerData.GameStats["total_wins"]++;
        }
        
        _playerData.GameStats["total_kills"] += kills;
        _playerData.GameStats["total_deaths"] += deaths;
        
        // Calculate XP
        int xpGained = CalculateMatchXP(won, kills, deaths, score);
        AddExperience(xpGained);
        
        // Add credits
        int creditsGained = score + (won ? 200 : 50);
        AddCredits(creditsGained);
        
        SaveGame();
    }
    
    private int CalculateMatchXP(bool won, int kills, int deaths, int score)
    {
        int baseXP = 100;
        int killXP = kills * 25;
        int scoreXP = score / 10;
        int winBonus = won ? 200 : 0;
        int survivalBonus = (kills > deaths) ? 100 : 0;
        
        return baseXP + killXP + scoreXP + winBonus + survivalBonus;
    }
    
    public void UnlockAchievement(string achievementID)
    {
        if (!_playerData.UnlockedAchievements.Contains(achievementID))
        {
            _playerData.UnlockedAchievements.Add(achievementID);
            
            // Award achievement rewards
            AwardAchievementReward(achievementID);
            
            SaveGame();
            
            // Show achievement notification
            UIManager.Instance.ShowAchievement(achievementID);
        }
    }
    
    private void AwardAchievementReward(string achievementID)
    {
        switch (achievementID)
        {
            case "first_blood":
                AddCredits(500);
                break;
            case "killing_spree":
                AddMechaParts(50);
                break;
            case "win_streak":
                AddCredits(1000);
                break;
            case "headshot_master":
                UnlockCosmetic("skin_golden");
                break;
        }
    }
    
    public PlayerData GetPlayerData() => _playerData;
    
    public int GetPlayerLevel() => _playerData.PlayerLevel;
    public int GetPlayerXP() => _playerData.Experience;
    public int GetPlayerCredits() => _playerData.Credits;
    public int GetPlayerMechaParts() => _playerData.MechaParts;
    
    public bool IsWeaponUnlocked(string weaponID)
    {
        return _playerData.Weapons.ContainsKey(weaponID) && 
               _playerData.Weapons[weaponID].IsUnlocked;
    }
    
    public bool IsMechaUnlocked(string mechaID)
    {
        return _playerData.Mechas.ContainsKey(mechaID) && 
               _playerData.Mechas[mechaID].IsUnlocked;
    }
    
    public bool IsCosmeticUnlocked(string cosmeticID)
    {
        return _playerData.Cosmetics.ContainsKey(cosmeticID) && 
               _playerData.Cosmetics[cosmeticID].IsUnlocked;
    }
    
    public void EquipCosmetic(string cosmeticID)
    {
        // Unequip all cosmetics of same type
        foreach (var cosmetic in _playerData.Cosmetics.Values)
        {
            cosmetic.IsEquipped = false;
        }
        
        // Equip new cosmetic
        if (_playerData.Cosmetics.ContainsKey(cosmeticID))
        {
            _playerData.Cosmetics[cosmeticID].IsEquipped = true;
        }
        
        SaveGame();
    }
    
    public SettingsData GetSettings() => _playerData.Settings;
    
    public void UpdateSettings(SettingsData settings)
    {
        _playerData.Settings = settings;
        SaveGame();
    }
    
    public void ResetProgress()
    {
        // Confirm before resetting
        _playerData = new PlayerData();
        InitializeDefaultData();
        SaveGame();
    }
    
    private void OnApplicationQuit()
    {
        SaveGame();
    }
    
    private void OnApplicationPause(bool pauseStatus)
    {
        if (pauseStatus)
        {
            SaveGame();
        }
    }
}// UIManager.cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using TMPro;

public class UIManager : MonoBehaviour
{
    public static UIManager Instance { get; private set; }
    
    [Header("Main Menu")]
    [SerializeField] private GameObject _mainMenuPanel;
    [SerializeField] private Button _playButton;
    [SerializeField] private Button _customizeButton;
    [SerializeField] private Button _shopButton;
    [SerializeField] private Button _settingsButton;
    [SerializeField] private Button _quitButton;
    
    [Header("Play Menu")]
    [SerializeField] private GameObject _playMenuPanel;
    [SerializeField] private Button _onlineButton;
    [SerializeField] private Button _offlineButton;
    [SerializeField] private Button _trainingButton;
    [SerializeField] private Button _backButton;
    
    [Header("Online Menu")]
    [SerializeField] private GameObject _onlineMenuPanel;
    [SerializeField] private Button _quickMatchButton;
    [SerializeField] private Button _createRoomButton;
    [SerializeField] private Button _joinRoomButton;
    [SerializeField] private TMP_InputField _roomCodeInput;
    [SerializeField] private TextMeshProUGUI _playerCountText;
    
    [Header("Lobby")]
    [SerializeField] private GameObject _lobbyPanel;
    [SerializeField] private Transform _playerListContent;
    [SerializeField] private TextMeshProUGUI _roomCodeText;
    [SerializeField] private Button _startGameButton;
    [SerializeField] private Button _leaveLobbyButton;
    [SerializeField] private Dropdown _gameModeDropdown;
    [SerializeField] private Dropdown _mapDropdown;
    
    [Header("In-Game HUD")]
    [SerializeField] private GameObject _hudPanel;
    [SerializeField] private TextMeshProUGUI _healthText;
    [SerializeField] private TextMeshProUGUI _armorText;
    [SerializeField] private TextMeshProUGUI _ammoText;
    [SerializeField] private TextMeshProUGUI _reserveAmmoText;
    [SerializeField] private TextMeshProUGUI _weaponNameText;
    [SerializeField] private Slider _healthSlider;
    [SerializeField] private Slider _armorSlider;
    [SerializeField] private Slider _energySlider;
    [SerializeField] private Slider _reloadSlider;
    [SerializeField] private Image _crosshair;
    [SerializeField] private GameObject _scopeOverlay;
    [SerializeField] private TextMeshProUGUI _killFeedText;
    [SerializeField] private Transform _killFeedContent;
    [SerializeField] private TextMeshProUGUI _timerText;
    [SerializeField] private TextMeshProUGUI _scoreText;
    
    [Header("Scoreboard")]
    [SerializeField] private GameObject _scoreboardPanel;
    [SerializeField] private Transform _scoreboardContent;
    [SerializeField] private GameObject _playerScorePrefab;
    
    [Header("Death Screen")]
    [SerializeField] private GameObject _deathPanel;
    [SerializeField] private TextMeshProUGUI _killerText;
    [SerializeField] private TextMeshProUGUI _respawnTimerText;
    
    [Header("Match Results")]
    [SerializeField] private GameObject _resultsPanel;
    [SerializeField] private TextMeshProUGUI _resultTitle;
    [SerializeField] private Transform _resultsContent;
    [SerializeField] private TextMeshProUGUI _xpGainedText;
    [SerializeField] private TextMeshProUGUI _creditsGainedText;
    
    [Header("Customization")]
    [SerializeField] private GameObject _customizePanel;
    [SerializeField] private GameObject _mechaCustomizeTab;
    [SerializeField] private GameObject _weaponCustomizeTab;
    [SerializeField] private GameObject _cosmeticCustomizeTab;
    [SerializeField] private Transform _mechaGrid;
    [SerializeField] private Transform _weaponGrid;
    [SerializeField] private Transform _cosmeticGrid;
    [SerializeField] private GameObject _itemPrefab;
    
    [Header("Shop")]
    [SerializeField] private GameObject _shopPanel;
    [SerializeField] private TextMeshProUGUI _creditsText;
    [SerializeField] private TextMeshProUGUI _mechaPartsText;
    [SerializeField] private Transform _shopItemsGrid;
    
    [Header("Settings")]
    [SerializeField] private GameObject _settingsPanel;
    [SerializeField] private Slider _masterVolumeSlider;
    [SerializeField] private Slider _musicVolumeSlider;
    [SerializeField] private Slider _sfxVolumeSlider;
    [SerializeField] private Slider _sensitivitySlider;
    [SerializeField] private Toggle _invertYToggle;
    [SerializeField] private Dropdown _graphicsDropdown;
    [SerializeField] private Button _applyButton;
    [SerializeField] private Button _resetButton;
    
    [Header("Notifications")]
    [SerializeField] private GameObject _notificationPanel;
    [SerializeField] private TextMeshProUGUI _notificationText;
    [SerializeField] private float _notificationDuration = 3f;
    
    [Header("Mobile Controls")]
    [SerializeField] private GameObject _mobileControlsPanel;
    [SerializeField] private Joystick _moveJoystick;
    [SerializeField] private Joystick _lookJoystick;
    [SerializeField] private Button _jumpButton;
    [SerializeField] private Button _crouchButton;
    [SerializeField] private Button _reloadButton;
    [SerializeField] private Button _fireButton;
    [SerializeField] private Button _aimButton;
    [SerializeField] private Button _weapon1Button;
    [SerializeField] private Button _weapon2Button;
    [SerializeField] private Button _weapon3Button;
    [SerializeField] private Button _weapon4Button;
    
    private Queue<string> _killFeedQueue = new();
    private Coroutine _killFeedCoroutine;
    private Coroutine _notificationCoroutine;
    private bool _isScoreboardVisible;
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    private void Start()
    {
        InitializeUI();
        SetupEventListeners();
        UpdatePlayerInfo();
        
        // Show main menu
        ShowMainMenu();
        
        // Set mobile controls for mobile platforms
        SetMobileControls(Application.isMobilePlatform);
    }
    
    private void InitializeUI()
    {
        // Hide all panels initially
        _mainMenuPanel.SetActive(false);
        _playMenuPanel.SetActive(false);
        _onlineMenuPanel.SetActive(false);
        _lobbyPanel.SetActive(false);
        _hudPanel.SetActive(false);
        _deathPanel.SetActive(false);
        _resultsPanel.SetActive(false);
        _customizePanel.SetActive(false);
        _shopPanel.SetActive(false);
        _settingsPanel.SetActive(false);
        _notificationPanel.SetActive(false);
        _mobileControlsPanel.SetActive(false);
        
        // Load settings
        LoadSettings();
    }
    
    private void SetupEventListeners()
    {
        // Main Menu
        _playButton.onClick.AddListener(ShowPlayMenu);
        _customizeButton.onClick.AddListener(ShowCustomizeMenu);
        _shopButton.onClick.AddListener(ShowShop);
        _settingsButton.onClick.AddListener(ShowSettings);
        _quitButton.onClick.AddListener(QuitGame);
        
        // Play Menu
        _onlineButton.onClick.AddListener(ShowOnlineMenu);
        _offlineButton.onClick.AddListener(StartOfflineGame);
        _trainingButton.onClick.AddListener(StartTraining);
        _backButton.onClick.AddListener(ShowMainMenu);
        
        // Online Menu
        _quickMatchButton.onClick.AddListener(FindQuickMatch);
        _createRoomButton.onClick.AddListener(CreateRoom);
        _joinRoomButton.onClick.AddListener(JoinRoom);
        
        // Lobby
        _startGameButton.onClick.AddListener(StartGameFromLobby);
        _leaveLobbyButton.onClick.AddListener(LeaveLobby);
        
        // Settings
        _applyButton.onClick.AddListener(ApplySettings);
        _resetButton.onClick.AddListener(ResetSettings);
        
        // Mobile Controls
        if (Application.isMobilePlatform)
        {
            SetupMobileControls();
        }
    }
    
    #region Menu Navigation
    
    public void ShowMainMenu()
    {
        HideAllPanels();
        _mainMenuPanel.SetActive(true);
        UpdatePlayerInfo();
    }
    
    public void ShowPlayMenu()
    {
        HideAllPanels();
        _playMenuPanel.SetActive(true);
    }
    
    public void ShowOnlineMenu()
    {
        HideAllPanels();
        _onlineMenuPanel.SetActive(true);
        UpdatePlayerCount();
    }
    
    public void ShowLobby(string roomCode, bool isHost)
    {
        HideAllPanels();
        _lobbyPanel.SetActive(true);
        _roomCodeText.text = $"Room Code: {roomCode}";
        _startGameButton.gameObject.SetActive(isHost);
    }
    
    public void ShowHUD()
    {
        HideAllPanels();
        _hudPanel.SetActive(true);
        
        if (Application.isMobilePlatform)
        {
            _mobileControlsPanel.SetActive(true);
        }
    }
    
    public void ShowDeathScreen(string killerName, string weapon)
    {
        _deathPanel.SetActive(true);
        _killerText.text = $"Killed by {killerName}\nwith {weapon}";
        
        StartCoroutine(RespawnCountdown());
    }
    
    private IEnumerator RespawnCountdown()
    {
        float respawnTime = 5f;
        
        while (respawnTime > 0)
        {
            _respawnTimerText.text = $"Respawning in {respawnTime:F1}";
            respawnTime -= Time.deltaTime;
            yield return null;
        }
        
        _deathPanel.SetActive(false);
    }
    
    public void ShowMatchResults(Dictionary<int, PlayerData> players, object winner)
    {
        HideAllPanels();
        _resultsPanel.SetActive(true);
        
        // Clear previous results
        foreach (Transform child in _resultsContent)
        {
            Destroy(child.gameObject);
        }
        
        // Set result title
        if (winner is int winnerID)
        {
            _resultTitle.text = $"Player {winnerID} Wins!";
        }
        else if (winner is Team winningTeam)
        {
            _resultTitle.text = $"{winningTeam} Team Wins!";
        }
        
        // Add player results
        foreach (var player in players.Values)
        {
            GameObject resultEntry = Instantiate(_playerScorePrefab, _resultsContent);
            resultEntry.GetComponent<PlayerResultEntry>().Setup(
                player.PlayerID,
                player.Kills,
                player.Deaths,
                player.Score,
                player.Team
            );
        }
        
        // Calculate rewards
        int xpGained = 250;
        int creditsGained = 500;
        
        _xpGainedText.text = $"+{xpGained} XP";
        _creditsGainedText.text = $"+{creditsGained} Credits";
        
        // Save rewards
        SaveManager.Instance.AddExperience(xpGained);
        SaveManager.Instance.AddCredits(creditsGained);
    }
    
    public void ShowCustomizeMenu()
    {
        HideAllPanels();
        _customizePanel.SetActive(true);
        
        LoadCustomizationItems();
    }
    
    public void ShowShop()
    {
        HideAllPanels();
        _shopPanel.SetActive(true);
        
        UpdateShopItems();
        UpdateCurrencyDisplay();
    }
    
    public void ShowSettings()
    {
        HideAllPanels();
        _settingsPanel.SetActive(true);
        
        LoadSettingsToUI();
    }
    
    private void HideAllPanels()
    {
        _mainMenuPanel.SetActive(false);
        _playMenuPanel.SetActive(false);
        _onlineMenuPanel.SetActive(false);
        _lobbyPanel.SetActive(false);
        _hudPanel.SetActive(false);
        _deathPanel.SetActive(false);
        _resultsPanel.SetActive(false);
        _customizePanel.SetActive(false);
        _shopPanel.SetActive(false);
        _settingsPanel.SetActive(false);
        _mobileControlsPanel.SetActive(false);
    }
    
    #endregion
    
    #region Gameplay UI Updates
    
    public void UpdateHealth(float current, float max)
    {
        _healthText.text = $"HP: {Mathf.CeilToInt(current)}/{Mathf.CeilToInt(max)}";
        _healthSlider.value = current / max;
        
        // Color change based on health
        Color healthColor = Color.Lerp(Color.red, Color.green, current / max);
        _healthSlider.fillRect.GetComponent<Image>().color = healthColor;
    }
    
    public void UpdateArmor(float current, float max)
    {
        _armorText.text = $"ARMOR: {Mathf.CeilToInt(current)}/{Mathf.CeilToInt(max)}";
        _armorSlider.value = current / max;
    }
    
    public void UpdateEnergy(float current, float max)
    {
        _energySlider.value = current / max;
    }
    
    public void UpdateAmmo(int current, int magazine, int reserve)
    {
        _ammoText.text = $"{current}/{magazine}";
        _reserveAmmoText.text = $"Reserve: {reserve}";
        
        // Low ammo warning
        _ammoText.color = current <= magazine * 0.2f ? Color.red : Color.white;
    }
    
    public void UpdateWeaponName(string name)
    {
        _weaponNameText.text = name;
    }
    
    public void UpdateReloadProgress(float progress)
    {
        _reloadSlider.gameObject.SetActive(progress > 0);
        _reloadSlider.value = progress;
    }
    
    public void SetScopeOverlay(bool visible)
    {
        _scopeOverlay.SetActive(visible);
        _crosshair.gameObject.SetActive(!visible);
    }
    
    public void SetCrosshairVisible(bool visible)
    {
        _crosshair.gameObject.SetActive(visible);
    }
    
    public void ShowHitMarker()
    {
        StartCoroutine(HitMarkerEffect());
    }
    
    private IEnumerator HitMarkerEffect()
    {
        _crosshair.color = Color.red;
        _crosshair.transform.localScale = Vector3.one * 1.2f;
        
        yield return new WaitForSeconds(0.1f);
        
        _crosshair.color = Color.white;
        _crosshair.transform.localScale = Vector3.one;
    }
    
    public void AddKillFeed(string killer, string victim, string weapon)
    {
        string killText = $"<color=#FFA500>{killer}</color> killed <color=#FF4444>{victim}</color> with {weapon}";
        _killFeedQueue.Enqueue(killText);
        
        if (_killFeedCoroutine == null)
        {
            _killFeedCoroutine = StartCoroutine(ProcessKillFeed());
        }
    }
    
    private IEnumerator ProcessKillFeed()
    {
        while (_killFeedQueue.Count > 0)
        {
            string killText = _killFeedQueue.Dequeue();
            
            // Create kill feed entry
            GameObject entry = new GameObject("KillFeedEntry");
            TextMeshProUGUI text = entry.AddComponent<TextMeshProUGUI>();
            text.text = killText;
            text.fontSize = 14;
            text.alignment = TextAlignmentOptions.Left;
            
            RectTransform rt = entry.GetComponent<RectTransform>();
            rt.SetParent(_killFeedContent);
            rt.localScale = Vector3.one;
            rt.sizeDelta = new Vector2(400, 30);
            
            // Add fade out effect
            StartCoroutine(FadeOutKillEntry(entry));
            
            // Limit entries
            if (_killFeedContent.childCount > 5)
            {
                Destroy(_killFeedContent.GetChild(0).gameObject);
            }
            
            yield return new WaitForSeconds(2f);
        }
        
        _killFeedCoroutine = null;
    }
    
    private IEnumerator FadeOutKillEntry(GameObject entry)
    {
        yield return new WaitForSeconds(4f);
        
        CanvasGroup group = entry.AddComponent<CanvasGroup>();
        float fadeTime = 1f;
        float elapsed = 0f;
        
        while (elapsed < fadeTime)
        {
            if (group != null)
            {
                group.alpha = 1f - (elapsed / fadeTime);
            }
            elapsed += Time.deltaTime;
            yield return null;
        }
        
        Destroy(entry);
    }
    
    public void UpdateScoreboard(Dictionary<int, PlayerData> players, Dictionary<Team, int> teamScores)
    {
        if (!_isScoreboardVisible) return;
        
        // Clear scoreboard
        foreach (Transform child in _scoreboardContent)
        {
            Destroy(child.gameObject);
        }
        
        // Add team scores for TDM
        if (teamScores != null && teamScores.Count > 0)
        {
            foreach (var teamScore in teamScores)
            {
                GameObject teamEntry = Instantiate(_playerScorePrefab, _scoreboardContent);
                teamEntry.GetComponent<PlayerResultEntry>().SetupTeam(teamScore.Key, teamScore.Value);
            }
        }
        
        // Add player scores
        foreach (var player in players.Values)
        {
            GameObject playerEntry = Instantiate(_playerScorePrefab, _scoreboardContent);
            playerEntry.GetComponent<PlayerResultEntry>().Setup(
                player.PlayerID,
                player.Kills,
                player.Deaths,
                player.Score,
                player.Team
            );
        }
    }
    
    public void ToggleScoreboard(bool show)
    {
        _isScoreboardVisible = show;
        _scoreboardPanel.SetActive(show);
        
        if (show)
        {
            // Update scoreboard data
            UpdateScoreboard(GameManager.Instance.GetPlayers(), GameManager.Instance.GetTeamScores());
        }
    }
    
    public void UpdateTimer(float time)
    {
        int minutes = Mathf.FloorToInt(time / 60);
        int seconds = Mathf.FloorToInt(time % 60);
        _timerText.text = $"{minutes:00}:{seconds:00}";
    }
    
    public void UpdateScore(int score)
    {
        _scoreText.text = $"Score: {score}";
    }
    
    #endregion
    
    #region Menu Functions
    
    private void FindQuickMatch()
    {
        NetworkManager.Instance.FindMatch(GameMode.FreeForAll);
        ShowNotification("Searching for match...");
    }
    
    private void CreateRoom()
    {
        string roomCode = GenerateRoomCode();
        NetworkManager.Instance.CreateRoom(roomCode, GameMode.FreeForAll, "DesertMechaZone", 10, false);
        ShowLobby(roomCode, true);
    }
    
    private void JoinRoom()
    {
        string roomCode = _roomCodeInput.text;
        if (string.IsNullOrEmpty(roomCode))
        {
            ShowNotification("Please enter a room code");
            return;
        }
        
        NetworkManager.Instance.JoinGame(roomCode);
        ShowNotification($"Joining room {roomCode}...");
    }
    
    private string GenerateRoomCode()
    {
        const string chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        char[] code = new char[6];
        
        for (int i = 0; i < 6; i++)
        {
            code[i] = chars[Random.Range(0, chars.Length)];
        }
        
        return new string(code);
    }
    
    private void StartGameFromLobby()
    {
        GameMode selectedMode = (GameMode)_gameModeDropdown.value;
        string selectedMap = _mapDropdown.options[_mapDropdown.value].text;
        
        GameManager.Instance.StartMatch(selectedMode, selectedMap);
        ShowHUD();
    }
    
    private void LeaveLobby()
    {
        NetworkManager.Instance.LeaveGame();
        ShowMainMenu();
    }
    
    private void StartOfflineGame()
    {
        GameManager.Instance.StartMatch(GameMode.Survival, "DesertMechaZone");
        ShowHUD();
    }
    
    private void StartTraining()
    {
        GameManager.Instance.StartMatch(GameMode.Training, "TrainingGrounds");
        ShowHUD();
    }
    
    #endregion
    
    #region Customization & Shop
    
    private void LoadCustomizationItems()
    {
        // Clear grids
        ClearGrid(_mechaGrid);
        ClearGrid(_weaponGrid);
        ClearGrid(_cosmeticGrid);
        
        // Load mechas
        PlayerData playerData = SaveManager.Instance.GetPlayerData();
        foreach (var mecha in playerData.Mechas.Values)
        {
            GameObject item = Instantiate(_itemPrefab, _mechaGrid);
            item.GetComponent<CustomizationItem>().Setup(
                mecha.MechaID,
                mecha.IsUnlocked,
                mecha.Level,
                OnCustomizationItemSelected
            );
        }
        
        // Load weapons
        foreach (var weapon in playerData.Weapons.Values)
        {
            GameObject item = Instantiate(_itemPrefab, _weaponGrid);
            item.GetComponent<CustomizationItem>().Setup(
                weapon.WeaponID,
                weapon.IsUnlocked,
                weapon.Level,
                OnCustomizationItemSelected
            );
        }
        
        // Load cosmetics
        foreach (var cosmetic in playerData.Cosmetics.Values)
        {
            GameObject item = Instantiate(_itemPrefab, _cosmeticGrid);
            item.GetComponent<CustomizationItem>().Setup(
                cosmetic.CosmeticID,
                cosmetic.IsUnlocked,
                0,
                OnCustomizationItemSelected
            );
        }
    }
    
    private void ClearGrid(Transform grid)
    {
        foreach (Transform child in grid)
        {
            Destroy(child.gameObject);
        }
    }
    
    private void OnCustomizationItemSelected(string itemID, CustomizationItemType type)
    {
        switch (type)
        {
            case CustomizationItemType.Mecha:
                // Equip mecha
                ShowNotification($"Equipped {itemID}");
                break;
                
            case CustomizationItemType.Weapon:
                // Equip weapon
                ShowNotification($"Equipped {itemID}");
                break;
                
            case CustomizationItemType.Cosmetic:
                // Equip cosmetic
                SaveManager.Instance.EquipCosmetic(itemID);
                ShowNotification($"Equipped {itemID}");
                break;
        }
    }
    
    private void UpdateShopItems()
    {
        ClearGrid(_shopItemsGrid);
        
        // Add shop items (this would come from a database or config file)
        List<ShopItem> shopItems = GetShopItems();
        
        foreach (var item in shopItems)
        {
            GameObject shopItem = Instantiate(_itemPrefab, _shopItemsGrid);
            shopItem.GetComponent<ShopItemUI>().Setup(
                item,
                OnShopItemPurchased
            );
        }
    }
    
    private List<ShopItem> GetShopItems()
    {
        // This would typically load from a config file or server
        return new List<ShopItem>
        {
            new ShopItem { ID = "mecha_heavy", Name = "Heavy Mecha", Description = "High armor, low mobility", 
                Price = 5000, Currency = CurrencyType.Credits, ItemType = ShopItemType.Mecha },
            new ShopItem { ID = "weapon_m82b", Name = "M82B Sniper", Description = "Anti-Mecha rifle", 
                Price = 3000, Currency = CurrencyType.Credits, ItemType = ShopItemType.Weapon },
            new ShopItem { ID = "skin_gold", Name = "Golden Skin", Description = "Premium cosmetic", 
                Price = 200, Currency = CurrencyType.MechaParts, ItemType = ShopItemType.Cosmetic }
        };
    }
    
    private void OnShopItemPurchased(ShopItem item)
    {
        bool purchaseSuccessful = false;
        
        if (item.Currency == CurrencyType.Credits)
        {
            purchaseSuccessful = SaveManager.Instance.SpendCredits(item.Price);
        }
        else
        {
            purchaseSuccessful = SaveManager.Instance.SpendMechaParts(item.Price);
        }
        
        if (purchaseSuccessful)
        {
            // Unlock the item
            switch (item.ItemType)
            {
                case ShopItemType.Mecha:
                    SaveManager.Instance.UnlockMecha(item.ID);
                    break;
                case ShopItemType.Weapon:
                    SaveManager.Instance.UnlockWeapon(item.ID);
                    break;
                case ShopItemType.Cosmetic:
                    SaveManager.Instance.UnlockCosmetic(item.ID);
                    break;
            }
            
            ShowNotification($"Purchased {item.Name}!");
            UpdateCurrencyDisplay();
            UpdateShopItems();
        }
        else
        {
            ShowNotification("Not enough currency!");
        }
    }
    
    private void UpdateCurrencyDisplay()
    {
        PlayerData data = SaveManager.Instance.GetPlayerData();
        _creditsText.text = $"Credits: {data.Credits}";
        _mechaPartsText.text = $"Mecha Parts: {data.MechaParts}";
    }
    
    private void UpdatePlayerInfo()
    {
        // Update player level, name, etc. in main menu
    }
    
    #endregion
    
    #region Settings
    
    private void LoadSettings()
    {
        SaveManager.SettingsData settings = SaveManager.Instance.GetSettings();
        
        // Apply audio settings
        AudioManager.Instance.SetMasterVolume(settings.MasterVolume);
        AudioManager.Instance.SetMusicVolume(settings.MusicVolume);
        AudioManager.Instance.SetSFXVolume(settings.SFXVolume);
        
        // Apply game settings
        InputManager.Instance.SetSensitivity(settings.Sensitivity);
        InputManager.Instance.SetInvertY(settings.InvertY);
        
        // Apply graphics settings
        QualitySettings.SetQualityLevel(settings.GraphicsQuality);
    }
    
    private void LoadSettingsToUI()
    {
        SaveManager.SettingsData settings = SaveManager.Instance.GetSettings();
        
        _masterVolumeSlider.value = settings.MasterVolume;
        _musicVolumeSlider.value = settings.MusicVolume;
        _sfxVolumeSlider.value = settings.SFXVolume;
        _sensitivitySlider.value = settings.Sensitivity;
        _invertYToggle.isOn = settings.InvertY;
        _graphicsDropdown.value = settings.GraphicsQuality;
    }
    
    private void ApplySettings()
    {
        SaveManager.SettingsData settings = new SaveManager.SettingsData
        {
            MasterVolume = _masterVolumeSlider.value,
            MusicVolume = _musicVolumeSlider.value,
            SFXVolume = _sfxVolumeSlider.value,
            Sensitivity = _sensitivitySlider.value,
            InvertY = _invertYToggle.isOn,
            GraphicsQuality = _graphicsDropdown.value
        };
        
        SaveManager.Instance.UpdateSettings(settings);
        LoadSettings();
        
        ShowNotification("Settings applied!");
    }
    
    private void ResetSettings()
    {
        SaveManager.SettingsData defaultSettings = new SaveManager.SettingsData();
        SaveManager.Instance.UpdateSettings(defaultSettings);
        LoadSettingsToUI();
        LoadSettings();
        
        ShowNotification("Settings reset to default!");
    }
    
    #endregion
    
    #region Mobile Controls
    
    private void SetMobileControls(bool enable)
    {
        _mobileControlsPanel.SetActive(enable);
    }
    
    private void SetupMobileControls()
    {
        _jumpButton.onClick.AddListener(() => InputManager.Instance.SetButtonDown("Jump"));
        _crouchButton.onClick.AddListener(() => InputManager.Instance.SetButton("Crouch", true));
        _reloadButton.onClick.AddListener(() => InputManager.Instance.SetButtonDown("Reload"));
        _fireButton.onClick.AddListener(() => InputManager.Instance.SetButton("Fire", true));
        _fireButton.onPointerUp.AddListener(() => InputManager.Instance.SetButton("Fire", false));
        _aimButton.onClick.AddListener(() => InputManager.Instance.SetButton("Aim", true));
        _aimButton.onPointerUp.AddListener(() => InputManager.Instance.SetButton("Aim", false));
        
        _weapon1Button.onClick.AddListener(() => InputManager.Instance.SetButtonDown("Weapon1"));
        _weapon2Button.onClick.AddListener(() => InputManager.Instance.SetButtonDown("Weapon2"));
        _weapon3Button.onClick.AddListener(() => InputManager.Instance.SetButtonDown("Weapon3"));
        _weapon4Button.onClick.AddListener(() => InputManager.Instance.SetButtonDown("Weapon4"));
        
        // Joystick events
        _moveJoystick.OnJoystickUpdate += (input) => InputManager.Instance.SetMovement(input);
        _lookJoystick.OnJoystickUpdate += (input) => InputManager.Instance.SetLook(input);
    }
    
    #endregion
    
    #region Utility
    
    public void ShowNotification(string message)
    {
        if (_notificationCoroutine != null)
        {
            StopCoroutine(_notificationCoroutine);
        }
        
        _notificationCoroutine = StartCoroutine(ShowNotificationCoroutine(message));
    }
    
    public void ShowAchievement(string achievementID)
    {
        string message = GetAchievementMessage(achievementID);
        ShowNotification($"Achievement Unlocked: {message}");
    }
    
    private string GetAchievementMessage(string achievementID)
    {
        return achievementID switch
        {
            "first_blood" => "First Blood - Get your first kill",
            "killing_spree" => "Killing Spree - Get 5 kills without dying",
            "win_streak" => "Win Streak - Win 3 matches in a row",
            "headshot_master" => "Headshot Master - Get 100 headshots",
            _ => "Achievement Unlocked"
        };
    }
    
    private IEnumerator ShowNotificationCoroutine(string message)
    {
        _notificationPanel.SetActive(true);
        _notificationText.text = message;
        
        yield return new WaitForSeconds(_notificationDuration);
        
        _notificationPanel.SetActive(false);
        _notificationCoroutine = null;
    }
    
    private void UpdatePlayerCount()
    {
        int playerCount = NetworkManager.Instance.GetPlayerCount();
        _playerCountText.text = $"Players Online: {playerCount}";
    }
    
    private void QuitGame()
    {
#if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
#else
        Application.Quit();
#endif
    }
    
    #endregion
}

public enum CustomizationItemType { Mecha, Weapon, Cosmetic }
public enum CurrencyType { Credits, MechaParts }
public enum ShopItemType { Mecha, Weapon, Cosmetic, Attachment }

[System.Serializable]
public class ShopItem
{
    public string ID;
    public string Name;
    public string Description;
    public int Price;
    public CurrencyType Currency;
    public ShopItemType ItemType;
    public Sprite Icon;
}// InputManager.cs
using System.Collections.Generic;
using UnityEngine;

public class InputManager : MonoBehaviour
{
    public static InputManager Instance { get; private set; }
    
    private Dictionary<string, bool> _buttons = new();
    private Dictionary<string, bool> _buttonsDown = new();
    private Dictionary<string, bool> _buttonsUp = new();
    
    private Vector2 _movementInput;
    private Vector2 _lookInput;
    
    // Settings
    private float _sensitivity = 1f;
    private bool _invertY = false;
    
    // Mobile
    private bool _isMobile;
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
            
            _isMobile = Application.isMobilePlatform;
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    private void Update()
    {
        if (!_isMobile)
        {
            UpdatePCInput();
        }
        
        ClearButtonStates();
    }
    
    private void UpdatePCInput()
    {
        // Movement
        _movementInput = new Vector2(
            (Input.GetKey(KeyCode.D) ? 1 : 0) - (Input.GetKey(KeyCode.A) ? 1 : 0),
            (Input.GetKey(KeyCode.W) ? 1 : 0) - (Input.GetKey(KeyCode.S) ? 1 : 0)
        );
        
        // Look
        _lookInput = new Vector2(
            Input.GetAxis("Mouse X"),
            Input.GetAxis("Mouse Y") * (_invertY ? 1 : -1)
        ) * _sensitivity;
        
        // Buttons
        SetButton("Jump", Input.GetKey(KeyCode.Space));
        SetButtonDown("Jump", Input.GetKeyDown(KeyCode.Space));
        SetButtonUp("Jump", Input.GetKeyUp(KeyCode.Space));
        
        SetButton("Fire", Input.GetMouseButton(0));
        SetButtonDown("Fire", Input.GetMouseButtonDown(0));
        SetButtonUp("Fire", Input.GetMouseButtonUp(0));
        
        SetButton("Aim", Input.GetMouseButton(1));
        SetButtonDown("Aim", Input.GetMouseButtonDown(1));
        SetButtonUp("Aim", Input.GetMouseButtonUp(1));
        
        SetButton("Reload", Input.GetKey(KeyCode.R));
        SetButtonDown("Reload", Input.GetKeyDown(KeyCode.R));
        
        SetButton("Crouch", Input.GetKey(KeyCode.C) || Input.GetKey(KeyCode.LeftControl));
        SetButtonDown("Crouch", Input.GetKeyDown(KeyCode.C) || Input.GetKeyDown(KeyCode.LeftControl));
        
        SetButton("Run", Input.GetKey(KeyCode.LeftShift));
        
        SetButtonDown("Weapon1", Input.GetKeyDown(KeyCode.Alpha1) || Input.GetKeyDown(KeyCode.Keypad1));
        SetButtonDown("Weapon2", Input.GetKeyDown(KeyCode.Alpha2) || Input.GetKeyDown(KeyCode.Keypad2));
        SetButtonDown("Weapon3", Input.GetKeyDown(KeyCode.Alpha3) || Input.GetKeyDown(KeyCode.Keypad3));
        SetButtonDown("Weapon4", Input.GetKeyDown(KeyCode.Alpha4) || Input.GetKeyDown(KeyCode.Keypad4));
        
        SetButtonDown("Ability1", Input.GetKeyDown(KeyCode.Q));
        SetButtonDown("Ability2", Input.GetKeyDown(KeyCode.E));
        SetButtonDown("Grenade", Input.GetKeyDown(KeyCode.G));
        
        SetButton("Scoreboard", Input.GetKey(KeyCode.Tab));
        SetButtonDown("Pause", Input.GetKeyDown(KeyCode.Escape));
    }
    
    private void ClearButtonStates()
    {
        _buttonsDown.Clear();
        _buttonsUp.Clear();
    }
    
    // Public API
    public Vector2 GetMovement()
    {
        return _movementInput;
    }
    
    public Vector2 GetLook()
    {
        return _lookInput;
    }
    
    public bool GetButton(string buttonName)
    {
        return _buttons.ContainsKey(buttonName) && _buttons[buttonName];
    }
    
    public bool GetButtonDown(string buttonName)
    {
        return _buttonsDown.ContainsKey(buttonName) && _buttonsDown[buttonName];
    }
    
    public bool GetButtonUp(string buttonName)
    {
        return _buttonsUp.ContainsKey(buttonName) && _buttonsUp[buttonName];
    }
    
    // Mobile input methods
    public void SetMovement(Vector2 input)
    {
        _movementInput = Vector2.ClampMagnitude(input, 1f);
    }
    
    public void SetLook(Vector2 input)
    {
        _lookInput = input * (_invertY ? -1 : 1) * _sensitivity;
    }
    
    public void SetButton(string buttonName, bool state)
    {
        _buttons[buttonName] = state;
    }
    
    public void SetButtonDown(string buttonName)
    {
        _buttons[buttonName] = true;
        _buttonsDown[buttonName] = true;
    }
    
    public void SetButtonUp(string buttonName)
    {
        _buttons[buttonName] = false;
        _buttonsUp[buttonName] = true;
    }
    
    // Settings
    public void SetSensitivity(float value)
    {
        _sensitivity = Mathf.Clamp(value, 0.1f, 5f);
    }
    
    public void SetInvertY(bool invert)
    {
        _invertY = invert;
    }
    
    public float GetSensitivity()
    {
        return _sensitivity;
    }
    
    public bool GetInvertY()
    {
        return _invertY;
    }
}
