# LAB 19 : Snake – Résolution détaillée du challenge Android

## 1. Présentation du lab

Ce rapport présente la résolution complète du challenge Android **Snake**, issu de **PwnSec CTF 2024 Mobile Hard**.

L’objectif du lab est de récupérer le flag caché dans l’application Android `snake.apk`.

L’application contient plusieurs protections :

- détection root ;
- détection Frida ;
- protections natives dans `libsnake.so` ;
- logique cachée dans une classe Java appelée `BigBoss` ;
- exploitation via un fichier YAML externe ;
- récupération du flag uniquement depuis `logcat`.

Le flag final obtenu est :

```text
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```

---

## 2. Environnement utilisé

Le lab a été réalisé dans l’environnement suivant :

```text
Système hôte : Windows
Terminal : PowerShell
Outils : ADB, Jadx-GUI, Apktool, zipalign, apksigner, keytool
Émulateur Android : Android API 28 recommandé
APK : snake.apk
Dossier de travail : C:\LAB19_SNAKE
```

Les outils principaux utilisés sont :

```text
Jadx-GUI     : analyse statique du code Java
Apktool      : décompilation et recompilation Smali
ADB          : interaction avec l’émulateur Android
zipalign     : alignement de l’APK patché
apksigner    : signature de l’APK patché
logcat       : récupération du flag
```

---

## 3. Objectif technique du challenge

Le challenge repose sur le flux suivant :

```text
Lancement de MainActivity avec un Intent spécifique
        |
        v
Extra Intent : SNAKE = BigBoss
        |
        v
Lecture du fichier Skull_Face.yml
        |
        v
Parsing du fichier avec SnakeYAML
        |
        v
Instanciation de la classe com.pwnsec.snake.BigBoss
        |
        v
Appel de la fonction native stringFromJNI()
        |
        v
Affichage du flag dans logcat
```

L’application ne donne pas directement le flag dans l’interface. Le flag est généré dynamiquement par la librairie native `libsnake.so`, puis affiché dans les logs Android.

---

## 4. Structure du dossier du lab

Le dossier de travail utilisé est :

```text
C:\LAB19_SNAKE
```

Organisation recommandée :

```text
LAB19_SNAKE/
│
├── rapport.md
└── captures/
    ├── 01_jadx_mainactivity.jpg
    ├── 02_jadx_bigboss.jpg
    ├── 03_apktool_decompile.jpg
    ├── 04_search_root_detection.jpg
    ├── 05_smali_before_patch.jpg
    ├── 06_smali_after_patch.jpg
    ├── 07_apktool_build.jpg
    ├── 08_zipalign.jpg
    ├── 09_apksigner.jpg
    ├── 10_install_patched_apk.jpg
    ├── 11_create_sdcard_folder.jpg
    ├── 12_create_yaml_payload.jpg
    ├── 13_push_yaml_payload.jpg
    ├── 14_app_result.jpg
    └── 15_logcat_flag.jpg
```



## 5. Analyse statique avec Jadx-GUI

L’APK a d’abord été ouvert avec **Jadx-GUI** afin de comprendre la logique interne de l’application.

Le package principal identifié est :

```text
com.pwnsec.snake
```

Les deux classes les plus importantes sont :

```text
MainActivity
BigBoss
```

---

## 6. Analyse de MainActivity

Dans `MainActivity`, on observe une méthode importante qui récupère l’Intent envoyé à l’application.

Le code vérifie la présence de l’extra suivant :

```text
SNAKE
```

La valeur attendue est :

```text
BigBoss
```

Cela signifie que l’application doit être lancée avec la commande suivante :

```powershell
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```

Dans la même méthode, l’application cherche ensuite un fichier YAML nommé :

```text
Skull_Face.yml
```

Ce fichier est lu depuis le stockage externe de l’émulateur.

Dans le code Jadx, on observe aussi l’utilisation de SnakeYAML avec une instruction de type :

```java
Yaml yaml = new Yaml();
Object obj = yaml.load(fileInputStream);
```

Cette partie est importante, car elle montre que le fichier YAML peut être utilisé pour instancier une classe Java.
<p align="center">
<img width="756" height="485" alt="01_jadx_mainactivity" src="https://github.com/user-attachments/assets/cb556954-441e-4b35-9c49-2dcfd6066e1f" />
</p>

---

## 7. Analyse de la classe BigBoss

La classe `BigBoss` est la cible principale du challenge.

Dans cette classe, on observe le chargement de la librairie native :

```java
System.loadLibrary("snake");
```

Cela correspond à la librairie native suivante :

```text
libsnake.so
```

La classe contient aussi un constructeur qui prend une chaîne de caractères en paramètre :

```java
public BigBoss(String str)
```

Dans ce constructeur, la méthode native suivante est appelée :

```java
stringFromJNI(str)
```

Ensuite, le résultat est converti depuis l’hexadécimal vers l’ASCII, puis affiché dans les logs Android avec le tag :

```text
BigBoss
```

La chaîne attendue pour déclencher la logique est :

```text
Snaaaaaaaaaaaaaake
```

Le principe est donc d’instancier la classe `BigBoss` avec cette chaîne via un fichier YAML.
<p align="center">
<img width="907" height="726" alt="02_jadx_bigboss" src="https://github.com/user-attachments/assets/79a56445-5cd9-44fd-98fb-77ee3ead487c" />
</p>

---

## 8. Décompilation de l’APK avec Apktool

Pour modifier l’application, l’APK a été décompilé avec Apktool.

Commande utilisée :

```powershell
apktool d .\snake.apk -o .\snake_smali
```

Cette commande génère le dossier suivant :

```text
snake_smali
```

Ce dossier contient :

```text
AndroidManifest.xml
smali/
lib/
res/
apktool.yml
```
<p align="center">
<img width="957" height="351" alt="03_apktool_decompile" src="https://github.com/user-attachments/assets/f9d29427-f47e-486a-a6cc-fa14bef6fa60" />
</p>

---

## 9. Recherche des protections root

Après la décompilation, une recherche a été effectuée dans les fichiers Smali pour identifier les protections root.

Commande utilisée :

```powershell
Get-ChildItem .\snake_smali -Recurse -Filter *.smali | Select-String -Pattern "root|su|Rooted|Superuser|test-keys" -CaseSensitive:$false
```

La recherche a permis de trouver plusieurs chaînes liées à la détection root :

```text
/sbin/su
/system/bin/su
/system/xbin/su
Rooted
Root detected
```

Ces éléments se trouvent principalement dans :

```text
snake_smali\smali\com\pwnsec\snake\MainActivity.smali
```
<p align="center">
<img width="807" height="295" alt="04_search_root_detection" src="https://github.com/user-attachments/assets/5677b50e-2677-49b3-83ae-dcc0cd37854e" />
</p>

---

## 10. Analyse de la méthode isDeviceRooted avant patch

Dans `MainActivity.smali`, la méthode suivante est responsable de la détection root :

```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
```

Avant modification, cette méthode appelait plusieurs fonctions de détection :

```text
checkForDangerousBinaries()
checkForRootManagementApps()
checkForWritableSystem()
checkForRootShell()
```

Ces fonctions vérifient notamment :

- la présence de binaires `su` ;
- la présence d’applications de gestion root ;
- l’écriture dans `/system` ;
- l’exécution d’un shell root.
<p align="center">
<img width="907" height="848" alt="05_smali_before_patch" src="https://github.com/user-attachments/assets/0570adff-c830-45ce-9e1c-e5d2c83274a7" />
</p>

---

## 11. Patch Smali de la détection root

Pour contourner la détection root, la méthode `isDeviceRooted()` a été modifiée afin de retourner toujours `false`.

Code Smali après patch :

```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1

    const/4 v0, 0x0
    return v0
.end method
```

Explication :

```text
const/4 v0, 0x0  => met false dans v0
return v0        => retourne false
```

Ainsi, l’application considère que l’appareil n’est pas rooté.
<p align="center">
<img width="842" height="322" alt="06_smali_after_patch" src="https://github.com/user-attachments/assets/af7512d8-3234-444b-a1cc-fcc00a210e4c" />
</p>

---

## 12. Neutralisation des fermetures forcées

Une recherche a aussi été faite pour identifier les appels qui ferment l’application :

```powershell
Get-ChildItem .\snake_smali -Recurse -Filter *.smali | Select-String -Pattern "System;->exit|Process;->killProcess|finish\(\)V" -CaseSensitive:$false
```

Les lignes importantes trouvées dans `MainActivity.smali` sont :

```text
invoke-virtual {p0}, Landroid/app/Activity;->finish()V
invoke-static {v0}, Ljava/lang/System;->exit(I)V
```

Ces instructions peuvent fermer l’application si une protection détecte un environnement non autorisé.

Elles ont été neutralisées en les remplaçant par :

```smali
nop
nop
```

Cela permet d’éviter une fermeture immédiate de l’application.

---

## 13. Recompilation de l’APK patché

Après modification du code Smali, l’APK a été recompilé.

Commande utilisée :

```powershell
apktool b .\snake_smali -o .\snake_patched_unsigned.apk
```

Résultat attendu :

```text
Built apk into: .\snake_patched_unsigned.apk
```
<p align="center">
<img width="922" height="238" alt="07_apktool_build" src="https://github.com/user-attachments/assets/a90db2b1-7286-48ef-9ab7-87ce72283139" />
</p>

---

## 14. Alignement de l’APK avec zipalign

Avant la signature, l’APK patché a été aligné avec `zipalign`.

Commande utilisée :

```powershell
& $zipalign -p -f 4 .\snake_patched_unsigned.apk .\snake_patched_aligned.apk
```

Fichier généré :

```text
snake_patched_aligned.apk
```
<p align="center">
<img width="936" height="416" alt="08_zipalign" src="https://github.com/user-attachments/assets/48bb529c-8004-4304-9700-b50224f12538" />
</p>

---

## 15. Création du keystore

Un keystore a été créé pour signer l’APK patché.

Commande utilisée :

```powershell
keytool -genkeypair -v -keystore .\snake-key.jks -alias snakekey -keyalg RSA -keysize 2048 -validity 10000
```

Informations saisies :

```text
Nom : ikram
Unité organisationnelle : gcdste
Entreprise : ensa
Ville : casablanca
Province : casablanca
Code pays : ma
```

Fichier généré :

```text
snake-key.jks
```

---

## 16. Signature de l’APK patché

L’APK aligné a ensuite été signé avec `apksigner`.

Commande utilisée :

```powershell
& $apksigner sign --ks .\snake-key.jks --ks-key-alias snakekey --out .\snake_patched_signed.apk .\snake_patched_aligned.apk
```

Fichier final généré :

```text
snake_patched_signed.apk
```
<p align="center">
<img width="961" height="508" alt="09_apksigner" src="https://github.com/user-attachments/assets/266b4dde-aebb-45ef-9e37-10343e656712" />
</p>

---

## 17. Installation de l’APK patché

L’ancienne version de l’application a été désinstallée :

```powershell
& $adb uninstall com.pwnsec.snake
```

Puis l’APK patché a été installé :

```powershell
& $adb install .\snake_patched_signed.apk
```

Résultat obtenu :

```text
Success
```
<p align="center">
<img width="767" height="188" alt="10_install_patched_apk" src="https://github.com/user-attachments/assets/7a17cf4b-d95d-4efb-b3fb-9f53c4909f43" />
</p>
---

## 18. Création du dossier externe attendu

L’application attend un fichier YAML dans un dossier du stockage externe.

Le dossier a été créé avec ADB :

```powershell
& $adb shell mkdir -p /sdcard/Snake
```

Vérification :

```powershell
& $adb shell ls -l /sdcard
```
<p align="center">
<img width="932" height="128" alt="11_create_sdcard_folder" src="https://github.com/user-attachments/assets/5d3ea6cd-d361-424a-8aa2-107df2f28087" />
</p>

---

## 19. Création du fichier YAML malveillant

Le fichier YAML utilisé dans ce lab est :

```text
Skull_Face.yml
```

Il contient le payload suivant :

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```

Commande utilisée pour créer le fichier :

```powershell
Set-Content -NoNewline -Path .\Skull_Face.yml -Value '!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]'
```

Vérification du contenu :

```powershell
Get-Content .\Skull_Face.yml
```

Résultat obtenu :

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```
<p align="center">
<img width="950" height="147" alt="12_create_yaml_payload" src="https://github.com/user-attachments/assets/79452992-8544-4289-9f09-120f1cb73f2d" />
</p>

---

## 20. Envoi du fichier YAML vers l’émulateur

Le fichier `Skull_Face.yml` a été transféré vers le dossier attendu par l’application.

Commande utilisée :

```powershell
& $adb push .\Skull_Face.yml /sdcard/Snake/Skull_Face.yml
```

Vérification :

```powershell
& $adb shell ls -l /sdcard/Snake
& $adb shell cat /sdcard/Snake/Skull_Face.yml
```

Résultat obtenu :

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```
<p align="center">
<img width="937" height="267" alt="13_push_yaml_payload" src="https://github.com/user-attachments/assets/ff6113b0-5fcf-45d8-b707-044b8d02a730" />
</p>

---

## 21. Lancement de l’application avec l’Intent requis

Pour déclencher la logique du challenge, il faut lancer `MainActivity` avec l’extra Intent suivant :

```text
SNAKE = BigBoss
```

Commande utilisée :

```powershell
& $adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```

Cette commande déclenche :

```text
MainActivity
   |
   v
Vérification de l'extra SNAKE
   |
   v
Lecture de Skull_Face.yml
   |
   v
Parsing SnakeYAML
   |
   v
Instanciation de BigBoss
   |
   v
Appel à stringFromJNI()
```

Après exécution, l’application affiche un écran avec le message :

```text
Here's to you Boss
```
<p align="center">
<img width="320" alt="14_app_result" src="https://github.com/user-attachments/assets/a8af1c42-e365-4088-9865-0aec269b756d" />
</p>
---

## 22. Récupération du flag dans logcat

Le flag n’est pas affiché directement dans l’interface graphique. Il est affiché dans les logs Android.

Commande utilisée :

```powershell
& $adb logcat -d | Select-String -Pattern "PWNSEC{"
```

Résultat obtenu :

```text
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```
<p align="center">
<img width="875" height="121" alt="15_logcat_flag" src="https://github.com/user-attachments/assets/f089a3be-d1d7-437d-9119-acc73dda3a12" />
</p>

---

## 23. Flag final

Le flag final récupéré est :

```text
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```

---





## 24. Conclusion

Ce lab a permis de réaliser une exploitation complète d’une application Android protégée.

Les compétences mises en pratique sont :

```text
Analyse statique Android avec Jadx
Décompilation APK avec Apktool
Lecture et modification du code Smali
Contournement de détection root
Recompilation et signature d’un APK patché
Création d’un payload YAML
Exploitation d’un parsing SnakeYAML non sécurisé
Interaction avec l’application via Intent ADB
Analyse des logs Android avec logcat
Récupération d’un flag généré par une librairie native JNI
```

Le challenge a été résolu avec succès. La preuve finale est le flag affiché dans `logcat` :

```text
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```

## 25. Auteure

Réalisé par : Ikram Laabouki

Module : Sécurité des Applications Mobiles

Établissement : ENSA Marrakech
