# LAB 18 — ViewModel & LiveData en Android
**Cours : Programmation Mobile – Android avec Java**
**Durée estimée : ~30 minutes**

---

## Objectifs

- Comprendre pourquoi les variables classiques sont perdues à la rotation d'écran
- Voir les limites de `onSaveInstanceState()`
- Maîtriser `ViewModel` + `LiveData` (lifecycle-aware)
- Appliquer le pattern MVVM recommandé par Google

---

## Structure du projet

```
ViewModelLiveDataDemoEnrichi/
├── app/
│   ├── build.gradle                          ← ajouter les dépendances ici
│   └── src/main/
│       ├── java/com/example/.../
│       │   ├── MainActivity.java             ← version finale (Partie 2)
│       │   └── CounterViewModel.java         ← nouveau fichier à créer
│       └── res/layout/
│           └── activity_main.xml             ← layout unique pour les 2 parties
```

---

## Étape 1 — Créer le projet

1. **File → New → New Project → Empty Activity**
2. Nom : `ViewModelLiveDataDemoEnrichi`
3. Language : **Java**
4. Minimum SDK : **API 24**
5. Cliquer **Finish**

---

## Étape 2 — Ajouter les dépendances

Ouvrir **`app/build.gradle`** et ajouter dans `dependencies { }` :

```groovy
def lifecycle_version = "2.10.0"

implementation "androidx.lifecycle:lifecycle-viewmodel:$lifecycle_version"
implementation "androidx.lifecycle:lifecycle-livedata:$lifecycle_version"
```

Cliquer **Sync Now** dans la barre jaune en haut.

> Sans ces dépendances, `ViewModel` et `LiveData` ne sont pas disponibles.

---

## Étape 3 — Layout (commun aux 2 parties)

**Fichier : `res/layout/activity_main.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center"
    android:orientation="vertical"
    android:padding="24dp">

    <TextView
        android:id="@+id/tvCount"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="0"
        android:textSize="80sp"
        android:textStyle="bold"
        android:layout_marginBottom="32dp"/>

    <Button
        android:id="@+id/btnIncrement"
        android:layout_width="200dp"
        android:layout_height="wrap_content"
        android:text="INCRÉMENTER"
        android:layout_marginBottom="8dp"/>

    <Button
        android:id="@+id/btnDecrement"
        android:layout_width="200dp"
        android:layout_height="wrap_content"
        android:text="DÉCRÉMENTER"
        android:layout_marginBottom="8dp"/>

    <Button
        android:id="@+id/btnReset"
        android:layout_width="200dp"
        android:layout_height="wrap_content"
        android:text="RÉINITIALISER"/>

</LinearLayout>
```

---

## Partie 1 — Version SANS ViewModel (le problème)

> Objectif : voir le bug → le compteur repart à 0 à la rotation.

**Fichier : `java/.../MainActivity.java`** *(version classique)*

```java
package com.example.viewmodellivedatademoenrichi;

import android.os.Bundle;
import android.widget.Button;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {

    // Variable classique → PERDUE à la rotation !
    private int count = 0;
    private TextView tvCount;
    private Button btnIncrement, btnDecrement, btnReset;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        tvCount      = findViewById(R.id.tvCount);
        btnIncrement = findViewById(R.id.btnIncrement);
        btnDecrement = findViewById(R.id.btnDecrement);
        btnReset     = findViewById(R.id.btnReset);

        // Restauration manuelle (ancienne méthode, limitée)
        if (savedInstanceState != null) {
            count = savedInstanceState.getInt("count_key", 0);
        }
        updateUI();

        btnIncrement.setOnClickListener(v -> { count++; updateUI(); });
        btnDecrement.setOnClickListener(v -> { count--; updateUI(); });
        btnReset.setOnClickListener(v    -> { count = 0; updateUI(); });
    }

    private void updateUI() {
        tvCount.setText(String.valueOf(count));
    }

    // Survie partielle à la rotation (primitifs uniquement)
    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putInt("count_key", count);
        // ⚠ Limitation : ne fonctionne pas pour objets complexes, threads, etc.
    }
}
```

**Test Partie 1 :**
- Lancer l'app → incrémenter 10 fois
- Rotation : **Ctrl + F11** (émulateur) ou tourner le téléphone
- ➡ Compteur revient à 0 → c'est le problème à résoudre !

---

## Partie 2 — Version AVEC ViewModel + LiveData (la solution)

### Fichier 1 : `CounterViewModel.java`

> Créer ce fichier dans le même package que `MainActivity.java`

```java
package com.example.viewmodellivedatademoenrichi;

import androidx.lifecycle.LiveData;
import androidx.lifecycle.MutableLiveData;
import androidx.lifecycle.ViewModel;

public class CounterViewModel extends ViewModel {

    // MutableLiveData = modifiable uniquement depuis le ViewModel
    private final MutableLiveData<Integer> countLiveData = new MutableLiveData<>();

    public CounterViewModel() {
        // Valeur initiale — appelée UNE SEULE FOIS (pas à chaque rotation)
        countLiveData.setValue(0);
    }

    public void increment() {
        Integer current = countLiveData.getValue();
        if (current != null) {
            countLiveData.setValue(current + 1); // met à jour tous les observers
        }
    }

    public void decrement() {
        Integer current = countLiveData.getValue();
        if (current != null) {
            countLiveData.setValue(current - 1);
        }
    }

    public void reset() {
        countLiveData.setValue(0);
    }

    // Exposé en lecture seule à l'Activity → bonne pratique MVVM
    public LiveData<Integer> getCount() {
        return countLiveData;
    }

    // BONUS 1 : incrémenter depuis un thread background (postValue = safe)
    public void incrementFromBackground() {
        new Thread(() -> {
            Integer current = countLiveData.getValue();
            if (current != null) {
                countLiveData.postValue(current + 1); // safe depuis n'importe quel thread
            }
        }).start();
    }
}
```

---

### Fichier 2 : `MainActivity.java` *(version finale — remplace la Partie 1)*

```java
package com.example.viewmodellivedatademoenrichi;

import android.os.Bundle;
import android.widget.Button;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;
import androidx.lifecycle.Observer;
import androidx.lifecycle.ViewModelProvider;

public class MainActivity extends AppCompatActivity {

    private CounterViewModel viewModel;
    private TextView tvCount;
    private Button btnIncrement, btnDecrement, btnReset;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        tvCount      = findViewById(R.id.tvCount);
        btnIncrement = findViewById(R.id.btnIncrement);
        btnDecrement = findViewById(R.id.btnDecrement);
        btnReset     = findViewById(R.id.btnReset);

        // 1. Récupération (ou création) du ViewModel
        //    → lié au Lifecycle de cette Activity, survit à la rotation
        viewModel = new ViewModelProvider(this).get(CounterViewModel.class);

        // 2. Observation du LiveData (lifecycle-aware)
        //    → l'observer n'est appelé QUE si l'Activity est STARTED ou RESUMED
        //    → supprimé automatiquement quand l'Activity est détruite → zéro leak
        viewModel.getCount().observe(this, new Observer<Integer>() {
            @Override
            public void onChanged(Integer newCount) {
                tvCount.setText(String.valueOf(newCount)); // mise à jour automatique !
            }
        });

        // 3. Les boutons appellent uniquement le ViewModel (séparation View / Logique)
        btnIncrement.setOnClickListener(v -> viewModel.increment());
        btnDecrement.setOnClickListener(v -> viewModel.decrement());
        btnReset.setOnClickListener(v    -> viewModel.reset());

        // BONUS 1 — test postValue depuis un thread background :
        // btnIncrement.setOnClickListener(v -> viewModel.incrementFromBackground());
    }
}
```

> `onSaveInstanceState` n'est plus nécessaire — le ViewModel gère tout.

---

## Étape 4 — Tests (obligatoires)

| # | Test | Résultat attendu |
|---|------|-----------------|
| 1 | Incrémenter 15 fois → **Ctrl+F11** | Compteur intact après rotation ✅ |
| 2 | Rotation multiple (3-4 fois) | Compteur toujours correct ✅ |
| 3 | Rotation + changement de thème | Données conservées ✅ |
| 4 | Commenter `observe()` → tester | UI ne se met plus à jour ✅ |

**Test process death (avancé) :**
```bash
adb shell am kill com.example.viewmodellivedatademoenrichi
```
Relancer l'app → le compteur reste (tant que le processus n'est pas tué par le système).

---

## Tableau comparatif

| Critère | Partie 1 (classique) | Partie 2 (ViewModel + LiveData) |
|---------|---------------------|--------------------------------|
| Survie rotation | Non | Oui (ViewModelStore) |
| Mise à jour UI automatique | Manuelle | Oui (Observer) |
| Gestion thread principal | Risque de crash | `setValue` / `postValue` sécurisé |
| Code propre (MVVM) | Mélangé | Séparé (ViewModel = logique) |
| Objets complexes | Non | Oui |
| Lifecycle-aware | Non | Oui (zéro crash, zéro memory leak) |

---

## Concepts clés à retenir

| Concept | Rôle |
|---------|------|
| `ViewModel` | Survit à la rotation, contient la logique |
| `MutableLiveData` | Modifiable depuis le ViewModel uniquement |
| `LiveData` | Lecture seule pour l'Activity (sécurité) |
| `setValue()` | Mise à jour depuis le **thread principal** |
| `postValue()` | Mise à jour depuis un **thread background** |
| `observe(this, ...)` | S'abonne aux changements, lifecycle-aware |
| `ViewModelProvider` | Crée ou récupère le ViewModel existant |

---

## Bonus 2 — SavedStateHandle (persistance après kill processus)

Modifer `CounterViewModel.java` pour survivre au process death :

```java
public class CounterViewModel extends ViewModel {

    private final SavedStateHandle savedStateHandle;
    private final MutableLiveData<Integer> countLiveData = new MutableLiveData<>();

    public CounterViewModel(SavedStateHandle handle) {
        this.savedStateHandle = handle;
        Integer saved = savedStateHandle.get("count");
        countLiveData.setValue(saved != null ? saved : 0);
    }

    public void increment() {
        Integer current = countLiveData.getValue();
        if (current != null) {
            int newVal = current + 1;
            countLiveData.setValue(newVal);
            savedStateHandle.set("count", newVal); // persisté même après kill
        }
    }

    // ... reste des méthodes identiques
}
```

Et dans `MainActivity.java`, changer l'initialisation :
```java
viewModel = new ViewModelProvider(this,
    SavedStateViewModelFactory.from(this))
    .get(CounterViewModel.class);
```

---

## Checklist finale

- [ ] Projet créé avec Java, API 24
- [ ] Dépendances `lifecycle-viewmodel` et `lifecycle-livedata` 2.10.0 ajoutées
- [ ] Sync Gradle réussi (pas d'erreurs)
- [ ] `activity_main.xml` créé avec les 4 vues IDs
- [ ] `CounterViewModel.java` créé dans le bon package
- [ ] `MainActivity.java` utilise `ViewModelProvider` + `observe()`
- [ ] Test rotation : compteur conservé ✅
- [ ] Test commentaire `observe()` : UI ne se met plus à jour ✅

---

## Erreurs fréquentes et solutions

| Erreur | Cause | Solution |
|--------|-------|----------|
| `Cannot resolve symbol 'ViewModel'` | Dépendances manquantes | Ajouter `lifecycle-viewmodel` dans `build.gradle` |
| `Cannot resolve symbol 'LiveData'` | Dépendances manquantes | Ajouter `lifecycle-livedata` dans `build.gradle` |
| Compteur toujours perdu à la rotation | Utilisation de `new CounterViewModel()` direct | Utiliser `new ViewModelProvider(this).get(...)` |
| UI ne se met pas à jour | `observe()` manquant ou commenté | Vérifier que `viewModel.getCount().observe(this, ...)` est présent dans `onCreate` |
| `NullPointerException` sur `getValue()` | LiveData pas initialisé | Vérifier `countLiveData.setValue(0)` dans le constructeur du ViewModel |

---

*Lab 18 — ViewModel & LiveData | Programmation Mobile Android*
