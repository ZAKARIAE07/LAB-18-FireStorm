# LAB-18-FireStorm
FireStorm — CTF Writeup

Niveau : Medium
Catégorie : Android Reverse Engineering
Techniques : Jadx · Frida · Firebase Authentication
Flag : PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}


🎯 Objectif
L'application Android contient une méthode Password() dans MainActivity qui génère un mot de passe pour s'authentifier sur Firebase. Cette méthode n'est jamais appelée dans le flux normal de l'application.
Le but est de :

Identifier la méthode cachée via reverse engineering avec Jadx
Forcer son exécution avec Frida pour extraire le mot de passe
S'authentifier sur Firebase et récupérer le flag


🛠️ Étape 1 — Préparation de l'environnement
Installer l'APK sur l'émulateur Android :
bashadb install FireStorm.apk
Vérifier que Frida est opérationnel et que le serveur Frida tourne sur l'appareil :
bashfrida-ps -U

🔍 Étape 2 — Analyse statique avec Jadx
Ouvrir l'APK dans Jadx-GUI et naviguer vers la classe principale.

Package : com.pwnsec.firestorm
Classe : MainActivity

La méthode clé découverte dans MainActivity :
javapublic String Password() {
    String s1 = getString(R.string.s1);
    String s2 = getString(R.string.s2);
    // Récupération d'autres chaînes depuis strings.xml

    // Appel à la fonction native dans libfirestorm.so
    String randomPart = generateRandomStrings(...);

    // Construction du mot de passe final
    String finalPassword = s1 + s2 + randomPart + ...;

    return finalPassword;
}

⚠️ Point clé : Cette méthode n'est appelée nulle part dans le code (ni dans onCreate, ni dans les listeners de boutons). C'est précisément ce qu'il faut exploiter.

Dans le fichier res/values/strings.xml, on trouve aussi :

Email Firebase : TK757567@pwnsec.xyz
La configuration complète Firebase (apiKey, databaseURL, etc.)


🪝 Étape 3 — Hooking dynamique avec Frida
Créer le fichier frida_firestorm.js :
javascriptJava.perform(function () {

    function getPassword() {
        console.log("[*] Début de la recherche d'instances de MainActivity...");

        Java.choose("com.pwnsec.firestorm.MainActivity", {

            onMatch: function (instance) {
                console.log("[+] Instance trouvée : " + instance);

                try {
                    var pass = instance.Password();
                    console.log("[!] MOT DE PASSE : " + pass);
                } catch (e) {
                    console.log("[-] Erreur : " + e);
                }
            },

            onComplete: function () {
                console.log("[*] Recherche terminée.");
            }
        });
    }

    console.log("[*] Script chargé. Attente de 3 secondes...");
    setTimeout(getPassword, 3000);
});
Lancer le script :
bashfrida -U -f com.pwnsec.firestorm -l frida_firestorm.js --no-pause
<img width="917" height="591" alt="1" src="https://github.com/user-attachments/assets/83fe202d-b42f-4ffa-91e5-eb30efc43bee" />

Le mot de passe s'affiche dans le terminal.

🔑 Mot de passe extrait : C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC


⚠️ Ce mot de passe est dynamique — il change à chaque lancement car la fonction native generateRandomStrings() dans libfirestorm.so produit un résultat différent.


☁️ Étape 4 — Configuration Firebase extraite de strings.xml
xml<string name="firebase_api_key">AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY</string>
<string name="firebase_email">TK757567@pwnsec.xyz</string>
<string name="firebase_database_url">https://firestorm-9d3db-default-rtdb.firebaseio.com</string>

🐍 Étape 5 — Récupération du flag avec Python
Créer le fichier get_flag.py :
pythonimport pyrebase

config = {
    "apiKey": "AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY",
    "authDomain": "firestorm-9d3db.firebaseapp.com",
    "databaseURL": "https://firestorm-9d3db-default-rtdb.firebaseio.com",
    "storageBucket": "firestorm-9d3db.appspot.com",
    "projectId": "firestorm-9d3db"
}

firebase = pyrebase.initialize_app(config)
auth = firebase.auth()

email = "TK757567@pwnsec.xyz"
password = "MOT_DE_PASSE_OBTENU_VIA_FRIDA"  # Remplacer par le mot de passe extrait

user = auth.sign_in_with_email_and_password(email, password)
print("[+] Connexion réussie !")

db = firebase.database()
flag_data = db.get(user["idToken"])
print("[!] FLAG RÉCUPÉRÉ :")
print(flag_data.val())
Exécuter :
bashpython get_flag.py
Résultat


<img width="756" height="206" alt="2" src="https://github.com/user-attachments/assets/3d20ab22-924a-4938-b533-bf668162f95a" />
🏁 Flag
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}

