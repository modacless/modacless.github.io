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
3. Management d'un match 
4. Tools d'analyse


## Mirror :

Pour ma première expérience dans le monde du multijoueur online, je voulais apprendre à utiliser une technologie gratuite, tout en me permettant d'acquérir une compréhension sur les API réseaux de haut niveaux. Mon choix c'est porté sur [Mirror](https://mirror-networking.com/), une API gratuite et open source.

Etant un jeux ou la vitesse, et la maitrise du personnage est la clef des mécanismes, nous avons ajouter une librairie, [Smooth Sync](https://forum.unity.com/threads/released-smooth-sync-smoothly-network-rigidbodies-and-transforms-while-reducing-bandwidth.486605/), ajoutant des scripts permettant une meilleurs coustomisation, et de meilleurs performance sur la position des objets "onlines".

## Lobby :

Pour la création du lobby, nous allons utiliser un template de NetworkManager que l'on va modifier.
L'objectif du loby dans notre jeu était de pouvoir voir le pseudo des autres joueurs, et de pouvoir se mettre ready, pour lancer la partie.

```c#
 private void CreateClientFromServer(NetworkConnection conn, MyNewNetworkAuthenticator.ClientConnectionMessage msg)
    {
        if (conn.clientOwnedObjects.Count < 1) // Débug quand le joueur se connecte à un server qui n'existe pas, et ensuite host
        {
            GameObject obj = Instantiate(lobbyPlayer);
            obj.GetComponent<LobbyPlayerLogic>().clientPseudo = msg.pseudo; //Créer le joueur loby 
            obj.transform.position = new Vector3(-1000, -1000, -1000);
            NetworkServer.AddPlayerForConnection(conn, obj);
            AddToServerArray(obj);
        }
        
    } //Spawn l'objet lobbyPlayer et configure le server

```





