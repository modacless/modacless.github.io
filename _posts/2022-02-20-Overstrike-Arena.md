---
layout: post
---

![theme logo](https://modacless.github.io/images/OverStrike/overstrikeArena.png)

---
<html>
<div style="text-align: center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/rDhEL9wTosI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>
</html>
---

# Introduction

Overstrike Arena est un fps multijoueur, en 2 contre 2, où les joueurs s'affrontent dans une arène avec un "punch", afin d'expulser leurs adversaires, et de marquer un but.

Page Steam : https://store.steampowered.com/app/1937550/Overstrike_Arena/

### Equipe

- Nicolas Capelier (Technical Game Designer && Porteur de vision)
- Théodore Labyt (Producer)
- Jean-Baptiste Oses (Lead Programmer)
- William Schmitt (Sound Designer)
- Lucas Deutschmann (Level Designer)
- Tom Watteau (Level Designer)
- Nicolas Perruchot(Marketing)
- Antoine Grugeon(3C Programmer && Designer)


### Mon travail

J'ai un peu touché à tout le domaine technique pour osa, j'ai travaillé sur du gameplay en collaboration avec le 3C designer, sur du réseau, sur du système (mise en place des manches, du "flag", des différentes équipes ...), et sur quelques "tools" pour aider mon équipe à produire plus d'éléments.

Si vous êtes intéressés par la partie réseaux et tools, j'explique un peu plus bas des bouts de codes que j'ai utilisés pour développer OSA.

---

# Technique

## Index

- [Mirror](#Mirror)
- [Lobby](#Lobby)
- [Match](#Match)
- [Outil et analyse](#Outil-et-analyse)


## Mirror

Pour ma première expérience dans le monde du multijoueur online, je voulais apprendre à utiliser une technologie gratuite, tout en me permettant d'acquérir une compréhension sur les API réseaux de haut niveau. Mon choix s'est porté sur [Mirror](https://mirror-networking.com/), une API gratuite et open source.

Etant un jeux ou la vitesse, et la maitrise du personnage sont les clefs des mécanismes, nous avons choisi une librairie, [Smooth Sync](https://forum.unity.com/threads/released-smooth-sync-smoothly-network-rigidbodies-and-transforms-while-reducing-bandwidth.486605/), ajoutant des scripts permettant une meilleure customisation, et de meilleurs performances sur la position des objets "online".

Le jeux va donc utiliser une structure server/client, où le client possède l'autorité, et le server est un joueur, qui joue comme les autres clients.

## Lobby

Pour la création du lobby, nous allons utiliser un template de NetworkManager que l'on va modifier. Le joueur décidant d'héberger, va attendre que les joueurs se connectent au server.

L'objectif du lobby dans notre jeu était de voir le pseudo des autres joueurs, de pouvoir changer d'équipe et de pouvoir se mettre "prêt", pour lancer la partie.

Quand un joueur se connecte au server, il lui envoi un message contenant son pseudo.
```c#
public override void OnClientConnect(NetworkConnection conn)//Quand le client se connecte envoit un message contenant le pseudo
    {
        base.OnClientConnect(conn);

        //Keep player pseudo
        playerPseudo = GetComponent<MyNewNetworkAuthenticator>().lobbyPseudo;

        //Send message to host with client pseudo
        MyNewNetworkAuthenticator.ClientConnectionMessage clientMsg = new MyNewNetworkAuthenticator.ClientConnectionMessage 
        {
            pseudo = GetComponent<MyNewNetworkAuthenticator>().lobbyPseudo
        };
        NetworkClient.Send(clientMsg);
    }
```

Le serveur le reçoit, il crée un gameObject représentant le joueur, et lui transmet le pseudo reçu (Les objets synchronisés ne pouvant être créés que par le serveur), il l'identifie ensuite en tant que joueur, et le lie à une connexion. Le serveur met ensuite à jour, sa liste interne des joueurs connectés au lobby, très utile pour vérifier quand les joueurs seront prêts.

```c#

public override void OnStartHost() {
        NetworkServer.RegisterHandler<MyNewNetworkAuthenticator.ClientConnectionMessage>(CreateClientFromServer, true);
        NetworkServer.RegisterHandler<MyNewNetworkAuthenticator.CreateClientPlayer>(CreatePlayer, true);
    }

 private void CreateClientFromServer(NetworkConnection conn, MyNewNetworkAuthenticator.ClientConnectionMessage msg)
    {
        if (conn.clientOwnedObjects.Count < 1) // Débug quand le joueur se connecte à un server qui n'existe pas, et ensuite host
        {
            GameObject obj = Instantiate(lobbyPlayer);
            obj.GetComponent<LobbyPlayerLogic>().clientPseudo = msg.pseudo; //Créer le joueur lobby 
            obj.transform.position = new Vector3(0, 0, 0); // Peut importe la position, une fois le jeu lancé, le joueur est placé au spawn
            NetworkServer.AddPlayerForConnection(conn, obj);
            AddToServerArray(obj);
        }
        
    } //Spawn l'objet lobbyPlayer et configure le server

```
Le gameobject représentant un joueur, possède un networkTransform, ça lui permet de mettre à jour sa position à travers le réseau.

![theme logo](images\OverStrike\CaptureLobbyPlayer.PNG)

Intéressons-nous aux fonctionnalités du "joueur lobby". 

Nous avons 3 variables déclarées comme des variables synchronisées.  Une variable synchronisée, peut-être changée depuis le serveur, et réplique ce changement à travers tous les clients. On peut y rajouter un "hook", pour pouvoir utiliser une fonction, à chaque fois que la variable change.

```c#
    [SyncVar(hook =nameof(setReadyUI))]
    public bool isReady = false;
    [SyncVar(hook = nameof(UpdateUsername))]
    public string clientPseudo;
    [SyncVar(hook = nameof(ChangeTeam))]
    public int team;
```

L'attribut Command, va déclencher la fonction de l'objet contrôlé par le client, sur l'objet contrôlé par ce client sur le serveur.
Donc, si l'on combine cet attribut avec les variables synchrones, on peut répliquer un changement de variable depuis un client, sur tous les autres clients.

Les fonctions ButtonLeft et ButtonRight sont rattachées à un unityEvent, sur des boutons, et se déclenchent quand un joueur appuie sur une flèche. Ainsi quand un client appuie sur un bouton, il demande au serveur d'exécuter cette fonction, ce qui va changer une variable synchronisée et appeler sa fonction "hook", qui va changer visuellement l'équipe du client, pour tous les joueurs.

```c#
[Command]
    public void ButonLeft()
    {
        if (!isReady)
        {
            if (team - 1 < 0)
            {
                team = 2;
            }
            else
            {
                team--;
            }
        }

    }
    [Command]
    public void ButonRight()
    {
        if (!isReady)
        {
            if (team + 1 > 2)
            {
                team = 0;
            }
            else
            {
                team++;
            }
        }
    }

    public void ChangeTeam(int oldValue,int newValue)
    {
        switch (newValue)
        {
            case 0:
                teamImage.sprite = omgegaImage;
                teamImage.color = omegaColor;
                teamName = TeamName.Red;
                break;
            case 1:
                teamImage.sprite = psiImage;
                teamImage.color = psyColor;
                teamName = TeamName.Blue;
                break;
            case 2:
                teamImage.sprite = spectatorImage;
                teamImage.color = spectatorColor;
                teamName = TeamName.Spectator;
                break;
        }
        if (hasAuthority)
        {
            serverManager.GetComponent<MyNewNetworkManager>().playerTeamName = teamName;
        }
        
    } //Syncronise l'ui

    [Command]
    public void CmdSetReady()
    {
        isReady = !isReady;
        serverManager.GetComponent<MyNewNetworkManager>().CheckIsReady();
    }
```

Afin de vérifier si chaque joueur est prêt, on va utiliser l'attribut command, pour vérifier si la variable isReady est "true" pour chaque client, si c'est le cas, on affiche un boutton "Start" , à l'host afin de pouvoir commencer une partie

```c#
     public bool CheckIsReady()
    {
        int redTeam = 0, blueTeam = 0;
        for (int i = 0; i < lobbyPlayerServer.Length; i++)
        {
            if (lobbyPlayerServer[i] != null)
            {
                if (lobbyPlayerServer[i].GetComponent<LobbyPlayerLogic>().teamName == LobbyPlayerLogic.TeamName.Blue)
                {
                    blueTeam++;
                }
                if (lobbyPlayerServer[i].GetComponent<LobbyPlayerLogic>().teamName == LobbyPlayerLogic.TeamName.Red)
                {
                    redTeam++;
                }

                if (!lobbyPlayerServer[i].GetComponent<LobbyPlayerLogic>().isReady)
                {
                    StartButton.SetActive(false);
                    return false;
                }
            }
        }

        if((redTeam == blueTeam) || (redTeam == 0 && blueTeam == 1) || (blueTeam == 0 && redTeam == 1) )
        {
            StartButton.SetActive(true);
            return true;
        }

        return false;

    }

```

On donc réussit à avoir notre joueur répliquer, avec un pseudo qui est lisible par tout le monde.

![theme logo](images\OverStrike\CaptureLobby.PNG)


## [Match](#Match)

Une fois que l'host lance la partie, chaque client doit charger la nouvelle map. Il doit attendre que chaque joueur finisse de charger le monde, pour commencer une partie. Mirror nous offre une booléenne pour connaître l'état du joueur `isReady`. Si la variable est égale à false, le joueur est en train de charger. Au chargement de la map, l'host va donc attendre que tous les clients aient fini de charger, avant "d'activer" le joueur, et de commencer une partie.

Les attributs Server et ServerCallback, vont permettre de spécifier des fonctions qui ne peuvent être lancées que par le serveur. Etant donné que c'est le serveur qui attend les joueurs, ces fonctions ne peuvent être qu'utilisées par lui.

On parcourt ici les valeurs d'un dictionnaire disponible côté serveur,  

```c#
 [ServerCallback]
    private bool AllClientAreReady()
    {
        foreach( NetworkConnectionToClient conn in NetworkServer.connections.Values)
        {
            if (!conn.isReady)
            {
                return false;
            }
        }
        return true;
    }

    [Server]
    private void ActivatePlayer()
    {
        foreach (NetworkConnectionToClient conn in NetworkServer.connections.Values)
        {
            foreach (NetworkIdentity idOwnedByClient in conn.clientOwnedObjects)
            {
                if (idOwnedByClient.gameObject.GetComponent<PlayerLogic>() != null)
                {
                    idOwnedByClient.gameObject.GetComponent<PlayerLogic>().RpcRespawn(conn,3f);
                }
            }
        }
    }

```

Notre jeu se divise en manches. 
Une manche commence après qu'un compteur se soit déclenché.

![theme logo](images\OverStrike\CaptureWait.PNG)

 Elle se termine quand une équipe marque un but.

![theme logo](images\OverStrike\CaptureEndManche.png)

Toute la gestion de la partie est géré grâce à mon GameObject "MatchManager".

![theme logo](images\OverStrike\CaptureGameManager.PNG)

Cet objet va gérer, le début et la fin de la partie, garder en mémoire les scores en bref manager le déroulement d'une partie entière.

```c#
    [Server]
    public void RpcEndGame(string text)
    {
        foreach (NetworkConnectionToClient conn in NetworkServer.connections.Values)
        {
            foreach (NetworkIdentity idOwnedByClient in conn.clientOwnedObjects)
            {
                if (idOwnedByClient.gameObject.GetComponent<PlayerLogic>() != null)
                {
                    idOwnedByClient.gameObject.GetComponent<PlayerLogic>().RpcEndGame(conn, text);
                }
            }
        }
    }

     [Server]
    private void ChangeUiPlayer(string text)
    {
        foreach (NetworkConnectionToClient conn in NetworkServer.connections.Values)
        {
            foreach (NetworkIdentity idOwnedByClient in conn.clientOwnedObjects)
            {
                if (idOwnedByClient.gameObject.GetComponent<PlayerLogic>() != null && !idOwnedByClient.gameObject.GetComponent<PlayerLogic>().isSpawning)
                {
                    //Creér une coroutile affichant l'ui, pui créer une nouvelle coroutine permettant de faire respawn les joueurs
                    idOwnedByClient.gameObject.GetComponent<PlayerLogic>().RpcShowGoal(conn,text);
                }
            }
        }
    }
```



On va utiliser des coroutines afin de créer une chronologie sur les actions liées au network, et de ne pas bloquer l'affichage de l'ui du joueur.

```c#
    [TargetRpc]
    public void RpcShowGoal(NetworkConnection conn,string text)
    {
        timerToStart = NetworkTime.time;
        if(respawnCor == null)
        {
            respawnCor = StartCoroutine(GoalMessageManager(text));
        }
            
        CmdShowScoreHud();
    }

    //Gère le respawn du joueur, situé dans le script du joueur
    public IEnumerator RespawnManager()
    {
        //Uniquement possible pour le joueur possédant l'autorithé
        if (hasAuthority)
        {
            //Stop le punch
            StopChargingPunch();
            selfMovement.FPVAnimator.StopAnimatePunch();

            actionExclusiveHud.SetActive(false);

            //Transition caméra au plafond
            if (overviewCameraPos == null)
                overviewCameraPos = GameObject.Find("OverviewCameraPosBlueSide").transform;
            overviewCamera.enabled = true;
            highlightCam.enabled = false;
            fpvCam.enabled = false;
            mainCam.enabled = false;
            
            //On a créer un Template Scène avec un Spawner, et un nombre d'enfant correspondant au des points dans l'espace
            Transform spawnPoint;
            spawnPoint = GameObject.FindWithTag("Spawner").transform.GetChild(spawnPosition);

            //Fade out de l'écran de chargement
            if (loadingScreen.color.a == 1)
            {
                loadingScreen.CrossFadeAlpha(0, 0.5f, false);
            }

            //Reset la position du joueur
            transform.position = spawnPoint.position; //Obligatoire, sinon ne trouve pas le spawner à la premirèe frame
            selfCollisionParent.transform.localRotation = spawnPoint.rotation;
            selfCamera.localRotation = spawnPoint.rotation;

            //Téléporte le joueur à travers le réseaux (Important, sinon peut laisser le joueur s'encastrer dans des murs)
            selfSmoothSync.teleportOwnedObjectFromOwner();
            selfCollsionSmoothSync.teleportOwnedObjectFromOwner();

            //Reset la rotation du joueur
            Quaternion startRot = selfCamera.localRotation;
            xRotation = startRot.eulerAngles.x;
            yRotation = startRot.eulerAngles.y;

            //Créer l'affichage du timer
            hudTextPlayer.gameObject.SetActive(true);

            //Lock caméra
            isSpawning = true;

            //Bloque le joueur et affiche le temps restant avant le respawn
            while (NetworkTime.time - timerToStart <= timerMaxToStart)
            {
                selfMovement.ResetVelocity();
                selfMovement.ResetVerticalVelocity();
                hudTextPlayer.text = System.Math.Ceiling(timerMaxToStart -(NetworkTime.time - timerToStart)).ToString();
                if (tryToRespawn)
                {
                    timerToStart = NetworkTime.time;
                    tryToRespawn = false;
                }
                overviewCamera.transform.position = overviewCameraPos.transform.position;
                overviewCamera.transform.rotation = overviewCameraPos.transform.rotation;

                yield return new WaitForEndOfFrame();

            }
            //La partie reprend pour le joueur
            roundStarted = true;
            hudTextPlayer.gameObject.SetActive(false);
            actionExclusiveHud.SetActive(true);

            //délock la camera
            isSpawning = false;
            overviewCamera.enabled = false;
            highlightCam.enabled = true;
            fpvCam.enabled = true;
            mainCam.enabled = true;

            //ajuste la rotation de la caméra
            xRotation = startRot.eulerAngles.x;
            yRotation = startRot.eulerAngles.y;

            //Initialise le joueur dans l'outil d'analyse
            if (GameObject.Find("Analytics") != null)
            {
                GameObject.Find("Analytics").GetComponent<PA_Position>().startWrite = true;
            }
            respawnCor = null;
        }
    }
```

## Outil et Analyse

Durant mon travail sur ce projet, j'ai dû créer quelques outils pour pouvoir répondre à certaines attentes des games designers. On voulait par exemple pouvoir analyser le déplacement des joueurs que l'on faisait tester.

J'ai donc créé poorAnalytics, un petit outil qui me permet de dessiner le déplacement des joueurs en fonction d'une manche.

![theme logo](images\OverStrike\PoorAnalytics.png)

![theme logo](images\OverStrike\PoorAnalytics2.png)

![theme logo](images\OverStrike\PoorAnalyticsUnity.PNG)

On peut décomposer cet outil en 2 scripts, un Writer, un Reader.

Le "writer", va écrire dans un fichier text, toutes les positions des objets stockés dans la liste analyticGameObjectPosition;
Afin de pouvoir utiliser ces données plus tard, je mets en place une architecture simple dans le fichier texte :

* `//` permet de définir que ce n'est pas une data brut
* `-` différencie chaque joueur
* `;` défini la fin de la ligne
* `|` permet de différencier les coordonnées (x,y,z)
* `++New Round++` Flag, qui permet au Reader de détecter un nouveau round

```c#
    public class PA_Position : MonoBehaviour
    {
        public MatchManager gameManager;
        public string fileToStorePosition;

        public float timeEachBreak;
        private float ownDeltaTime;

        public List<Transform> analyticGameObjectPosition;

        private bool onceInit = true;
        [HideInInspector]
        public bool startWrite = true;

        private void Start()
        {
            if (GameObject.Find("ServerManager").GetComponent<MyNewNetworkManager>().analyticsPath != string.Empty)
            {
                fileToStorePosition = GameObject.Find("ServerManager").GetComponent<MyNewNetworkManager>().analyticsPath;
            }
            else
            {
                gameObject.SetActive(false);
            }
            
        }

        void InitAnalytics()
        {
            StreamWriter writer = new StreamWriter(fileToStorePosition, false);
            writer.Write("// ");
            for (int i = 0; i < analyticGameObjectPosition.Count; i++)
            {
                writer.Write("Object " + i.ToString() + "-");
            }
            writer.WriteLine();
            writer.Close();
        }

        // Update is called once per frame
        void Update()
        {
            if (gameManager.startGame)
            {
                if (onceInit)
                {
                    onceInit = false;
                    InitAnalytics();
                }

                ownDeltaTime += Time.deltaTime;
                if (ownDeltaTime >= timeEachBreak && startWrite)
                {
                    WriteAllObjectPosition();
                    ownDeltaTime = 0f;
                }
            }
            
        }

        void OnApplicationQuit()
        {
            StreamWriter writer = new StreamWriter(fileToStorePosition, true);
            DateTime dt = DateTime.Now;
            writer.WriteLine("// " + dt.ToString() + " //");
            writer.Close();
        }

        private void WriteAllObjectPosition()
        {
            StreamWriter writer = new StreamWriter(fileToStorePosition, true);
            foreach(Transform objPosition in analyticGameObjectPosition)
            {
                writer.Write(objPosition.position.x.ToString() + "|" + objPosition.position.y.ToString() + "|" + objPosition.position.z.ToString() + ";");
            }
            writer.Write("\n");
            writer.Close();
        }

        public void WriteNewRound()
        {
            startWrite = false;
            StreamWriter writer = new StreamWriter(fileToStorePosition, true);
            writer.Write("++New Round++");
            writer.Write("\n");
            writer.Close();
        }
    }
```

Le "Reader", récupère le path du fichier texte, et grâce à un bouton (Draw in world), va lancer la fonction `LoadKeyPositions`, qui s'occupe de parser et de stocker les positions dans une liste 3 dimensions.
Chaque dimension correspondant à un des paramètres de la partie -> Quelle manche, Quel joueur, ses positions.


```c#
public List<List<List<Vector3>>> playersKeyPosition = new List<List<List<Vector3>>>();

 public void LoadKeyPositions()
    {

        playersKeyPosition.Clear();
        roundPointeur = 0;
        round = 0;
        try
        {
            for (int i = 0; i < 5; i++)
            {
                playersKeyPosition.Add(new List<List<Vector3>>());
            }

            StreamReader sr = new StreamReader(pathFilePositions);

            string firstLine = sr.ReadLine();
            string[] allPlayersInLine = firstLine.Split(new[] { '-' }, StringSplitOptions.RemoveEmptyEntries);
            
            //Colors add or remove
            if(colors.Count > allPlayersInLine.Length)
            {
                while(colors.Count != allPlayersInLine.Length)
                {
                    colors.RemoveAt(colors.Count-1);
                }
            }
            if (colors.Count < allPlayersInLine.Length)
            {
                while (colors.Count != allPlayersInLine.Length)
                {
                    colors.Add(new Color());
                }
            }

            foreach (string players in allPlayersInLine)
            {
                if(players.Length > 1)
                {
                    for(int i = 0; i < 5; i++)
                    {
                        playersKeyPosition[i].Add(new List<Vector3>());
                    }
                }
            }
            
            //Read and parse file position
            while (!sr.EndOfStream)
            {
                string line = sr.ReadLine();

                if (line.Contains("++"))
                {
                    roundPointeur++;
                }
                if (!line.Contains("//") && !line.Contains("++"))
                {
                    //Find each object
                    string[] playersPositions = line.Split(new[] { ';' }, StringSplitOptions.RemoveEmptyEntries);

                    //Parse position for an object
                    for(int i = 0; i < playersPositions.Length; i++)
                    {
                        string[] playerPosition = playersPositions[i].Split(new[] { '|' }, StringSplitOptions.RemoveEmptyEntries);
                        float _x = float.Parse(playerPosition[0]);
                        float _y = float.Parse(playerPosition[1]);
                        float _z = float.Parse(playerPosition[2]);

                        playersKeyPosition[roundPointeur][i].Add(new Vector3(_x, _y, _z));
                    }
                }
                
            }
            sr.Close();
        }catch(Exception ex)
        {
            Debug.Log(ex.ToString());
        }
    }
```

Une fois les données stockées, on va implémenter une fonction apportée par Monobehavior : OnDrawnGizmos.
Cette fonction va nous permettre de dessiner des formes simples directement dans la scène du moteur de jeux. Une fonction très utile pour le debugging ou l'analyse du jeu.
On va alors dessiner une ligne entre les coordonnées de chaque joueur en fonction de la manche, indiqué par un entier depuis l'inspector.

```c#

 void OnDrawGizmos()
    {

        //Gizmos.DrawLine(new Vector3(0,0,0), new Vector3(0, 100, 0));
        int j = 0;

        if(playersKeyPosition.Count > 0)
        {
            foreach (List<Vector3> KeyPositions in playersKeyPosition[round])
            {
                Gizmos.color = colors[j];
                if (KeyPositions.Count > 3) // sup to 3 because there is min 2 lines per files
                {
                    for (int i = 1; i < KeyPositions.Count; i++)
                    {
                        Gizmos.DrawLine(KeyPositions[i - 1], KeyPositions[i]);
                    }
                }
                j++;
            }
        }

    }
```

On utilise ici un fichier texte pour stocker les données afin de  pouvoir l'échanger, le stocker, le récupérer plus facilement. On peut aussi essayer d'automatiser l'envoi de ce fichier sur un server. À la fin d'une partie, l'hôte pourrait envoyer automatiquement le texte sur un server que l'on héberge, ça permettrait à mon équipe d'avoir facilement accès à toutes les données des joueurs ayant essayé le jeu online.

Il y a beaucoup de moyens pour améliorer cet outil, l'utilisation d'une liste 3d, n'est vraiment pas une bonne pratique en informatique par exemple. Un onceInit, totalement inutile, qui peut-être remplacé par la création d'une coroutine au start, qui se met en attente, tant que les conditions requises ne sont pas terminés.

