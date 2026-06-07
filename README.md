# Lab11 — Google Maps Android : Localisation temps réel et marqueurs dynamiques (Java)

**Auteur :** DOSSAH Yao Landry  
**Filière :** Génie CyberDéfense et Systèmes de Télécommunications Embarquées (GCDSTE)  
**Établissement :** ENSA Marrakech

---

## Contexte pédagogique

Ce laboratoire porte sur l'intégration de Google Maps dans une application Android Java. L'application **MapsActivity** affiche une carte interactive, demande la permission de localisation au runtime, écoute les mises à jour de position via `LocationManager`, place un marqueur unique qui se déplace en temps réel, et propose une boîte de dialogue pour activer le GPS si celui-ci est désactivé.

---

## Environnement

| Composant | Détail |
|-----------|--------|
| IDE | Android Studio |
| Langage | Java |
| Min SDK | API 24 |
| Template de départ | Google Maps Activity |
| API utilisée | Google Maps SDK for Android |
| Provider de localisation | `NETWORK_PROVIDER` (Wi-Fi/4G) |
| Permission requise | `ACCESS_FINE_LOCATION` (runtime, Android 6+) |

---

## Architecture du projet

```
MapsApp/
├── app/src/main/
│   ├── java/.../
│   │   └── MapsActivity.java        ← activité principale, toute la logique
│   ├── res/
│   │   ├── layout/
│   │   │   └── activity_maps.xml    ← fragment SupportMapFragment
│   │   └── values/
│   │       └── google_maps_api.xml  ← clé API (non versionnée)
│   └── AndroidManifest.xml          ← déclaration des permissions
└── build.gradle                     ← dépendance Maps SDK
```

---

## Flux d'exécution

```
Lancement de l'application
        |
        ↓
MapsActivity.onCreate()
        |
        ↓
SupportMapFragment.getMapAsync(this)
        |
        ↓
onMapReady(GoogleMap googleMap)
        |
        ├── Vérification permission ACCESS_FINE_LOCATION
        │       |
        │       ├── Permission accordée
        │       │       |
        │       │       ↓
        │       │   locationManager.requestLocationUpdates(NETWORK_PROVIDER)
        │       │       |
        │       │       ↓
        │       │   onLocationChanged(location)
        │       │       |
        │       │       ├── currentMarker == null → addMarker()
        │       │       └── currentMarker != null → setPosition()
        │       │               |
        │       │               ↓
        │       │           animateCamera(newLatLngZoom, 15f)
        │       │
        │       └── Permission refusée
        │               |
        │               ↓
        │           requestPermissions(..., 200)
        │               |
        │               ↓
        │           onRequestPermissionsResult()
        │               └── Accordée → onMapReady(mMap)
        │
        └── onProviderDisabled() → buildAlertMessageNoGps()
                                        |
                                        ├── "Yes" → Settings.ACTION_LOCATION_SOURCE_SETTINGS
                                        └── "No"  → dialog.cancel()
```

---

## Partie 1 — Configuration du projet et clé API

Le projet est créé depuis le template **Google Maps Activity** d'Android Studio, qui génère automatiquement `MapsActivity.java`, `activity_maps.xml` et `google_maps_api.xml`.

La clé API est générée depuis la Google Cloud Console en activant le **Maps SDK for Android**, puis placée dans `google_maps_api.xml` :

```xml
<string name="google_maps_key" templateMergeStrategy="preserve" translatable="false">
    AIzaSy...VOTRE_CLE
</string>
```

**Résultat observé :** au premier lancement sans clé, la carte affiche une grille grise avec le filigrane *"For development purposes only"*. Après insertion d'une clé valide avec Maps SDK activé, les tuiles de carte se chargent correctement.

---

## Partie 2 — Permissions

Deux permissions sont déclarées dans `AndroidManifest.xml` :

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.INTERNET" />
```

La déclaration dans le manifest seule ne suffit pas sur Android 6+. La permission `ACCESS_FINE_LOCATION` est dite *dangereuse* — elle doit être accordée explicitement par l'utilisateur au runtime. Sans cette étape, `requestLocationUpdates()` lève une `SecurityException`.

La vérification et la demande runtime sont gérées dans `onMapReady()` :

```java
boolean permissionGranted =
    ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION)
        == PackageManager.PERMISSION_GRANTED;

if (!permissionGranted) {
    ActivityCompat.requestPermissions(
        this,
        new String[]{Manifest.permission.ACCESS_FINE_LOCATION},
        200
    );
}
```

**Résultat observé :** au premier lancement, le dialogue système de permission s'affiche. Après accord, `onRequestPermissionsResult()` rappelle `onMapReady(mMap)` pour démarrer l'écoute de localisation sans redémarrer l'activité.

---

## Partie 3 — Écoute de la localisation (LocationManager)

`LocationManager` est le service Android qui expose les providers de localisation. Deux providers sont disponibles :

| Provider | Précision | Disponibilité |
|----------|-----------|---------------|
| `NETWORK_PROVIDER` | Moyenne (~50-100m) | Intérieur, rapide |
| `GPS_PROVIDER` | Haute (~5-10m) | Extérieur, lent |

Le lab utilise `NETWORK_PROVIDER` pour garantir des mises à jour même en intérieur :

```java
locationManager.requestLocationUpdates(
    LocationManager.NETWORK_PROVIDER,
    1000,   // intervalle minimum : 1 seconde
    50,     // distance minimum : 50 mètres
    locationListener
);
```

**Résultat observé :** sur émulateur, la position est simulée via *Extended Controls → Location*. Sur appareil réel, le provider réseau remonte la position en quelques secondes après activation du Wi-Fi ou de la 4G.

---

## Partie 4 — Marker unique dynamique

Pour éviter la pollution visuelle (un marker par update), un seul marker est maintenu et déplacé à chaque nouvelle position :

```java
private Marker currentMarker;

@Override
public void onLocationChanged(Location location) {
    LatLng pos = new LatLng(location.getLatitude(), location.getLongitude());

    if (currentMarker == null) {
        currentMarker = mMap.addMarker(
            new MarkerOptions().position(pos).title("Position actuelle")
        );
    } else {
        currentMarker.setPosition(pos);
    }

    mMap.animateCamera(CameraUpdateFactory.newLatLngZoom(pos, 15f));
}
```

`animateCamera` est préféré à `moveCamera` pour une transition fluide. Le niveau de zoom 15 correspond à l'échelle rue/quartier.

**Résultat observé :** à chaque changement de position simulé, le marker se déplace et la caméra suit avec animation. Aucun marker résiduel ne s'accumule sur la carte.

---

## Partie 5 — Dialogue GPS désactivé

`onProviderDisabled()` se déclenche si le provider est coupé pendant l'écoute. Une `AlertDialog` non annulable propose d'ouvrir les paramètres de localisation :

```java
private void buildAlertMessageNoGps() {
    new AlertDialog.Builder(this)
        .setMessage("Your GPS seems to be disabled, do you want to enable it?")
        .setCancelable(false)
        .setPositiveButton("Yes", (dialog, id) ->
            startActivity(new Intent(Settings.ACTION_LOCATION_SOURCE_SETTINGS)))
        .setNegativeButton("No", (dialog, id) -> dialog.cancel())
        .create()
        .show();
}
```

`setCancelable(false)` empêche la fermeture par tap extérieur — l'utilisateur doit faire un choix explicite.

**Résultat observé :** en désactivant la localisation depuis les paramètres pendant que l'app tourne, la boîte de dialogue apparaît immédiatement. Le bouton *Yes* redirige vers `Settings.ACTION_LOCATION_SOURCE_SETTINGS`.

---

## Erreurs fréquentes rencontrées

| Erreur | Cause | Solution |
|--------|-------|----------|
| Carte blanche / filigrane *"For development purposes only"* | Clé API invalide ou Maps SDK non activé | Vérifier la clé et activer Maps SDK for Android dans la Cloud Console |
| `SecurityException` sur `requestLocationUpdates` | Permission non vérifiée au runtime | Toujours appeler `checkSelfPermission` avant `requestLocationUpdates` |
| Aucun marker visible | Provider réseau désactivé ou permission refusée | Activer la localisation, vérifier les logs Logcat |
| Position ne change jamais | `minDistance = 50m` trop élevé pour les tests | Passer à `0` en développement pour déclencher à chaque update |
| Dialog GPS ne s'affiche pas | `onProviderDisabled` non déclenché sur certains providers | Vérifier l'état du GPS en amont dès `onMapReady` |

---

## Points clés retenus

- La permission `ACCESS_FINE_LOCATION` doit être vérifiée **et** demandée au runtime sur Android 6+ — la déclaration dans le manifest seul ne suffit pas.
- `NETWORK_PROVIDER` est préférable en environnement intérieur ou pour les tests ; `GPS_PROVIDER` est plus précis mais exige un ciel dégagé et un temps de fix plus long.
- Maintenir un seul `Marker` et appeler `setPosition()` est plus propre que d'accumuler un marker par update de position.
- `animateCamera` offre une meilleure expérience utilisateur que `moveCamera` pour les déplacements fréquents.
- `onRequestPermissionsResult()` est indispensable : sans lui, l'acceptation de la permission par l'utilisateur n'a aucun effet immédiat sur l'écoute de localisation.
- La clé API ne doit jamais être versionnée dans le dépôt Git — ajouter `google_maps_api.xml` au `.gitignore` en production.

---

*Lab réalisé dans le cadre du cours Développement Mobile — ENSA Marrakech, Filière GCDSTE*
