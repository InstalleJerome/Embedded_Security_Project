# Embedded_Security_Project
Project for the embedded security course at ECAM. The goal is to find the password that grants access to a protected asset.




# Power Analysis sur Arduino — Bypass d'authentification

## 1. Préparation de la cible

Premièrement, il fallait flasher l'Arduino avec le firmware nécessaire au projet. La commande suivante permet cela :

```bash
avrdude -v -patmega328p -carduino -PCOM11 -b115200 -Uflash:w:secure_sketch_v20251015.2.elf
```

`secure_sketch_v20251015.2` étant le firmware en question.

## 2. Attack Tree

<img width="1445" height="1398" alt="image" src="https://github.com/user-attachments/assets/64ed1ed9-916c-4bd9-88c8-9b0f9e99c335" />


## 3. Accès naïf

Pour commencer, on essaye de se connecter simplement en série à l'Arduino (avec PuTTY par exemple). Voici ce que nous renvoie l'Arduino :

*(image à insérer)*

Comme on peut le voir, cet accès n'est pas le bon, il faudra trouver l'accès « dérobé » pour pouvoir communiquer avec l'Arduino.

## 4. Analyseur logique

Afin de trouver les RX et TX utilisés par le firmware, on peut tester les différentes pins avec un analyseur logique et observer le résultat avec PulseView, ce qui nous permettra d'analyser le comportement de l'Arduino au démarrage. En arrivant sur la pin 11, on observe ceci :

*(image à insérer)*

On peut donc déduire que nous avons trouvé TX. Voici ce qui se passe lorsqu'on se connecte en série à cette pin :

*(image à insérer)*

On peut tester les différentes pins pour trouver RX en essayant de rentrer un message et regarder quand est-ce que le terminal réagit. Ici, on trouve la pin 10.

## 5. Préparation de l'attaque

**Power analysis to Timing analysis using ChipWhisperer.**

Afin de pouvoir attaquer l'Arduino via le ChipWhisperer, il fallait sortir l'ATmega de son socket et le connecter tel que le montre l'image ci-dessous :

*(image à insérer)*

Il faut encore effectuer quelques modifications :

- On connecte le **GND** et le **VREF** du ChipWhisperer (pins 2 et 8) au GND et 5V du montage.
- On connecte également les pins 10 et 12 du ChipWhisperer aux pins 16 et 17 de l'ATmega, ce qui correspond aux pins 10 et 11 de l'Arduino, donc à RX et TX utilisés pour communiquer.
- La **sonde B** est utilisée pour faire la mesure de la consommation lors de la vérification du mot de passe. On la connecte donc à un des emplacements « measure » du CW.
- Finalement, lors des tests avec l'analyseur logique, nous avons remarqué que la pin 13 de l'Arduino (pin 19 de l'ATmega) se met à l'état haut lors de la vérification du mot de passe. Nous allons donc utiliser cette pin comme trigger de la prise de mesure : on la connecte avec la **sonde A** à la pin 16 du CW.

## 6. Attaque

### 6.1 Préparation du ChipWhisperer

Le bloc suivant permet de créer les documents utiles au ChipWhisperer :

```python
# (code à insérer)
```

Le code suivant permet de se connecter au ChipWhisperer et d'envoyer un message au moment du boot, permettant ainsi de vérifier si la connexion entre l'ATmega et le CW est bonne et qu'on arrive à lui envoyer une requête :

```python
# (code à insérer)
```

Finalement, cette ligne permet d'utiliser le script de setup de base contenu dans le notebook.

### 6.2 Attaque

Afin de trouver le mot de passe, nous allons effectuer une Power Analysis. Le principe est simple : on essaye toutes les possibilités pour chaque caractère du mot de passe et on compare la moyenne de consommation lors de la vérification du mot de passe. Le caractère dont la consommation moyenne diffère de celle des autres est probablement le bon.

La fonction `cap_pass_trace` suivante permet de prendre une mesure avec le ChipWhisperer. Comme énoncé plus haut, cette mesure est effectuée grâce à la sonde B et représente la puissance consommée lors de la vérification du mot de passe :

```python
# (code à insérer)
```

Les points importants de cette fonction sont :

- **`scope.arm()`**, qui permet de mettre le CW en stand-by et d'attendre un flanc montant sur son trigger pour ensuite commencer une mesure avec `scope.capture()`.
- **`target.ser.write()`** permet d'envoyer notre tentative de mot de passe. Ne pas oublier le `"\n"`, sinon on ne fait qu'écrire les mots de passe à la suite sans jamais envoyer de proposition, et donc rester en état ARM.
- **`resync()`** permet de se resynchroniser entre les différentes prises de mesure et de vider le buffer pour avoir des mesures plus stables.

Une fois que nous avons cette fonction, il suffit de l'exécuter plusieurs fois sur tous les caractères possibles et pour toutes les positions possibles. Ici, on fait la moyenne de consommation sur 15 itérations, 15 étant un bon compromis entre précision et temps d'exécution.

**La boucle de travail est donc la suivante :**

1. On envoie un caractère 15 fois et on récupère la moyenne de consommation, pour chaque caractère ASCII existant.
2. On calcule la moyenne générale et on compare la moyenne de chaque caractère à cette moyenne globale.
3. Le caractère qui ressort le plus est le meilleur candidat.
4. On continue caractère par caractère jusqu'à ne plus avoir `"ACCESS DENIED"` en réponse.

Une fois le mot de passe trouvé, on obtient le hash et le salt, synonyme de hack réussi :

*(image à insérer)*

## 7. Contre-mesure

Cette attaque se basant sur la puissance consommée et donc le temps d'activation, il suffit de jouer sur ce paramètre pour déjouer l'attaque. En regardant le code en clair, on voit que la fonction qui s'occupe de garantir l'accès à la ressource est la fonction `early_exit` :

```cpp
// (code à insérer)
```

Cette fonction va s'arrêter dès qu'un caractère de la proposition diffère du vrai mot de passe. En faisant en sorte que l'entièreté du mot de passe soit vérifiée à chaque essai, on agit de deux manières :

- On augmente le temps de mesure sur chaque mot de passe.
- Étant donné que le temps de mesure sera constant pour tout mot de passe de même longueur, on ne pourra rien en déduire.

On peut voir que l'attaque ne fonctionne plus (le premier caractère trouvé devrait être `'f'`) :

*(image à insérer)*

## 8. CVSS

Le score de sévérité de l'attaque est de **4.1** (calculé grâce au calculateur [nvd.nist.gov/vuln-metrics/cvss/v4-calculator](https://nvd.nist.gov/vuln-metrics/cvss/v4-calculator)). Ce score est assez faible et est justifié par ces 3 points principaux :

- **Attack Vector: Physical** — Signifie qu'il faut une manipulation physique de la cible.
- **Attack Complexity: High** — Signifie qu'il faut en plus une manipulation pour la prise de mesure et un traitement de ces données. Ici, on fait des centaines de captures et une analyse dessus, le tout plusieurs fois, et non juste cliquer et avoir la réponse.
- **Attack Requirements: None** — Il ne faut pas de conditions spécifiques pour que l'attaque soit réussie.

Le vecteur complet est le suivant :

```
CVSS:4.0/AV:P/AC:H/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:N/SI:N/SA:N
```
