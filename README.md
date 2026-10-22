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

<img width="1445" height="1398" alt="mermaid-diagram-2026-10-22-235223" src="https://github.com/user-attachments/assets/2e8dd34c-d0ae-40cd-a01a-a6f1a663321d" />


## 3. Accès naïf

Pour commencer, on essaye de se connecter simplement en série à l'Arduino (avec PuTTY par exemple). Voici ce que nous renvoie l'Arduino :

<img width="540" height="71" alt="Capture d&#39;écran 2026-10-07 104619" src="https://github.com/user-attachments/assets/4331ffc7-449d-40f0-93f1-7d09aab7b196" />

Comme on peut le voir, cet accès n'est pas le bon, il faudra trouver l'accès « dérobé » pour pouvoir communiquer avec l'Arduino.

## 4. Analyseur logique

Afin de trouver les RX et TX utilisés par le firmware, on peut tester les différentes pins avec un analyseur logique et observer le résultat avec PulseView, ce qui nous permettra d'analyser le comportement de l'Arduino au démarrage. En arrivant sur la pin 11, on observe ceci :

<img width="1917" height="1017" alt="Capture d&#39;écran 2026-10-07 102909" src="https://github.com/user-attachments/assets/15a96cde-cb0a-4308-8388-552aa6b2becb" />

On peut donc déduire que nous avons trouvé TX. Voici ce qui se passe lorsqu'on se connecte en série à cette pin :

<img width="215" height="80" alt="Capture d&#39;écran 2026-10-20 094706" src="https://github.com/user-attachments/assets/4caded10-6313-46f8-8017-73868b5794b9" />

On peut tester les différentes pins pour trouver RX en essayant de rentrer un message et regarder quand est-ce que le terminal réagit. Ici, on trouve la pin 10. Ces pins vont donc nous permettre d'interagir avec l'arduino.

## 5. Préparation de l'attaque

**Power analysis to Timing analysis using ChipWhisperer.**

Afin de pouvoir attaquer l'Arduino via le ChipWhisperer, il fallait sortir l'ATmega de son socket et le connecter tel que le montre l'image ci-dessous :

<img width="1117" height="772" alt="Capture d&#39;écran 2026-10-22 141624" src="https://github.com/user-attachments/assets/1cb9aa3d-74e6-4326-8747-12ed9e1a2581" />

Il faut encore effectuer quelques modifications :

- On connecte le **GND** et le **VREF** du ChipWhisperer (pins 2 et 8) au GND et 5V du montage.
- On connecte également les pins 10 et 12 du ChipWhisperer aux pins 16 et 17 de l'ATmega, ce qui correspond aux pins 10 et 11 de l'Arduino, donc à RX et TX utilisés pour communiquer.
- La **sonde B** est utilisée pour faire la mesure de la consommation lors de la vérification du mot de passe. On la connecte donc à un des emplacements « measure » du ChipWhisperer.
- Finalement, lors des tests avec l'analyseur logique, nous avons remarqué que la pin 13 de l'Arduino (pin 19 de l'ATmega) se met à l'état haut lors de la vérification du mot de passe. Nous allons donc utiliser cette pin comme trigger de la prise de mesure : on la connecte avec la **sonde A** à la pin 16 du CW.

## 6. Attaque

### 6.1 Préparation du ChipWhisperer

Le bloc suivant permet de créer les documents utiles au ChipWhisperer :

```python
%%bash
cd ../chipwhisperer/firmware/mcu/stm32-gpio-tristate
make PLATFORM='CWNANO' CRYPTO_TARGET=NONE -j
```

Le code suivant permet de se connecter au ChipWhisperer et d'envoyer un message au moment du boot, permettant ainsi de vérifier si la connexion entre l'ATmega et le CW est bonne et qu'on arrive à lui envoyer une requête. Il permet aussi de définir la cible de l'attaque :

```python
import chipwhisperer as cw
 
scope = cw.scope()
target = cw.target(scope, cw.targets.SimpleSerial2)
target.baud = 9600
target.ser.write("hello")

print("Boot message:", target.ser.read(50))  
 
# Send a wrong password
target.ser.write(b"1234\n")
print("Response:", target.ser.read(50))
scope.dis()
```

Finalement, cette ligne permet d'utiliser le script de setup de base contenu dans le notebook.

```python
%run "../chipwhisperer/jupyter/Setup_Scripts/Setup_Generic.ipynb"
```

### 6.2 Attaque

Afin de trouver le mot de passe, nous allons effectuer une Power Analysis. Le principe est simple : on essaye toutes les possibilités pour chaque caractère ASCII possible du mot de passe et on compare la moyenne de consommation lors de la vérification du mot de passe. Le caractère dont la consommation moyenne diffère de celle des autres est probablement le bon.

La fonction `cap_pass_trace` suivante permet de prendre une mesure avec le ChipWhisperer. Comme énoncé plus haut, cette mesure est effectuée grâce à la sonde B et représente la puissance consommée lors de la vérification du mot de passe :

```python
def cap_pass_trace(pass_guess):
    resync()
    
    scope.arm()
    target.ser.write((pass_guess + "\n").encode())  # le \n déclenche le trigger
    
    ret = scope.capture()
    if ret:
        return None
    
    return scope.get_last_trace()
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

<img width="882" height="117" alt="Capture d&#39;écran 2026-10-20 094722" src="https://github.com/user-attachments/assets/b420a90a-f076-4eb2-b7c5-d9fac98e17e2" />


## 7. Contre-mesure

Cette attaque se basant sur la puissance consommée et donc le temps d'activation, il suffit de jouer sur ce paramètre pour la déjouer. En regardant le code en clair, on voit que la fonction qui s'occupe de garantir l'accès à la ressource est la fonction `early_exit_compare` :

```cpp
bool early_exit_compare(const char *attempt) {
  size_t pass_len = sizeof(PASSWORD) - 1; // exclude terminating NUL
  for (size_t i = 0; i < pass_len; ++i) {
    /*Serial.print("Character from password: '");
    Serial.print(PASSWORD[i]);
    Serial.print("', attempt: '");/home/dlh/ECAM/Cours/5MEO Embedded Security/Course/00/ES5T_00/resource/ES5_CM0_code_naif.ino
    Serial.print(attempt[i]);
    Serial.println("'");*/
    // if attempt ended early or characters differ -> fail immediately
    if (attempt[i] == '\0') return false;
    if (attempt[i] != PASSWORD[i]) return false;
    //delayMicroseconds(50);
  }
  // ensure attempt has no extra characters
  return true;
}
```

Cette fonction va s'arrêter dès qu'un caractère de la proposition diffère du vrai mot de passe. En faisant en sorte que l'entièreté du mot de passe soit vérifiée à chaque essai, on agit de deux manières :

- On augmente le temps de mesure sur chaque mot de passe.
- Étant donné que le temps de mesure sera constant pour tout mot de passe de même longueur, on ne pourra rien en déduire.

On peut voir que l'attaque ne fonctionne plus (le premier caractère trouvé devrait être `'f'`) :

<img width="525" height="330" alt="Capture d&#39;écran 2026-10-20 152037" src="https://github.com/user-attachments/assets/7c49abd3-2ceb-46e5-b3f7-b3445b82f397" />

## 8. CVSS

Le score de sévérité de l'attaque est de **4.1** (calculé grâce au calculateur [nvd.nist.gov/vuln-metrics/cvss/v4-calculator](https://nvd.nist.gov/vuln-metrics/cvss/v4-calculator)). Ce score est assez faible et est justifié par ces 3 points principaux :

- **Attack Vector: Physical** — Signifie qu'il faut une manipulation physique de la cible.
- **Attack Complexity: High** — Signifie qu'il faut en plus une manipulation pour la prise de mesure et un traitement de ces données. Ici, on fait des centaines de captures et une analyse dessus, le tout plusieurs fois, et non juste cliquer et avoir la réponse.
- **Attack Requirements: None** — Il ne faut pas de conditions spécifiques pour que l'attaque soit réussie.

Le vecteur complet est le suivant :

```
CVSS:4.0/AV:P/AC:H/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:N/SI:N/SA:N
```
Un des point d'amélioration principal serait de changer l'état des **Attack Requirements : Present**. Par exemple, imposer un certain temps entre chaque tentative et ne donc pas pouvoir faire de teste en boucle.
