---
layout: post
---

![theme logo](https://modacless.github.io/images/OverStrike/overstrikeArena.png)

---

## Introduction

Overstrike Arena est un fps multijoueur, en 2 contre 2, ou les joueurs s'affrontent dans une arène avec un "punch", afin d'expulser leurs adversaires, et de marquer un but.

### Equipes

- Nicolas Capelier (Technical Game Designer && Porteur de vision)
- Théodore Labyt (Producer)
- Jean-Baptiste Oses (Lead Programmer)
- William Schmitt (Sound Designer)
- Lucas Deutschmann (Level Designer)
- Tom Watteau (Level Designer)
- Nicolas Perruchot(Marketing)
- Antoine Grugeon(3C Programmer && Designer)

---

### Mon Rôle

## Techniques

1. Mirror, et smooth network Transform
2. Lobby 
3. Match
4. Tools d'analyse


## Mirror :

Pour ma première expérience dans le monde du multijoueur online, je voulais apprendre à utiliser une technologie gratuite, tout en me permettant d'acquérir une compréhension sur les API réseaux de haut niveaux. Mon choix s'est porté sur [Mirror](https://mirror-networking.com/), une API gratuite et open source.

Etant un jeux ou la vitesse, et la maitrise du personnage est la clef des mécanismes, nous avons ajouter une librairie, [Smooth Sync](https://forum.unity.com/threads/released-smooth-sync-smoothly-network-rigidbodies-and-transforms-while-reducing-bandwidth.486605/), ajoutant des scripts permettant une meilleurs coustomisation, et de meilleurs performance sur la position des objets "onlines".

Le jeux va donc utiliser une structure server/client, ou le client possède l'authorité, et le server est un joueur, qui joue comme les autres clients.

## Lobby :

Pour la création du lobby, nous allons utiliser un template de NetworkManager que l'on va modifier. Le joueur décidant d'héberger, va attendre que les joueurs se connectent au server.

L'objectif du lobby dans notre jeu était de pouvoir voir le pseudo des autres joueurs, de pouvoir changer d'équipe et de pouvoir se mettre "prêt", pour lancer la partie.

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

Le server le reçoit, il crée un gameObject représentant le joueur, et lui transmet le pseudo reçu (Les objects synchronisés ne pouvant être créer que par le serveur), il l'identifie ensuite en tant que joueur, et le lie à une connection. Le serveur met ensuite à jours, sa liste interne des joueurs connecté au lobby, très utile pour vérifier quand les joueurs seront prêts.

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

Intéressons nous aux fonctionnalité du "joueur lobby". 

Nous avons 3 variables déclarées comme des variables synchronisées.  Une variable synchronisée, peut-être changé depuis le server, et réplique ce changement à travers tous les clients. On peut y rajouter un "hook", pour pouvoir utiliser une fonction, à chaque fois que la variable change.

```c#
    [SyncVar(hook =nameof(setReadyUI))]
    public bool isReady = false;
    [SyncVar(hook = nameof(UpdateUsername))]
    public string clientPseudo;
    [SyncVar(hook = nameof(ChangeTeam))]
    public int team;
```

L'attribut Command, va déclencher la fonction de l'objet contrôlé par le client, sur l'object contrôlé par ce client sur le server.
Donc, si l'on combine cette attribut avec les variables syncrones, on peut répliquer un changement de variable depuis un client, sur tous les autres clients.

Les fonctions ButtonLeft et ButtonRight sont rattachés à un unityEvent, sur des boutons, et se déclenche quand un joueur appuie sur une flèche. Ainsi quand un client appuie sur un boutton, il demande au server d'exécuter cette fonction, ce qui va changer une variable synchronisée et appeler sa fonction "hook", qui va changer visuellement l'équipe du client, pour tous les joueurs.

```c#
[Command]
    public void ButtonLeft()
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
    public void ButtonRight()
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

Afin de vérifier si chaque joueur est prêts, on va utiliser l'attribut command, pour vérifier si la variable isReady est "true" pour chaque client, si c'est le cas, on affiche un boutton "Start" , à l'host affin de pouvoir commencer une partie

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


## Match

Notre jeux se divise en manches. 
Une manche commence après qu'un compteur ce soit déclenché.

![theme logo](images\OverStrike\CaptureWait.PNG)

 Elle se termine quand une équipe marque un but.

![theme logo](images\OverStrike\CaptureEndManche.png)
