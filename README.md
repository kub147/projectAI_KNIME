# 🧹 EIACD Project – Agentic AI dla usługi sprzątania

Projekt na zaliczenie przedmiotu **EIACD** (Agentic AI for Deployment of ML Models).  
Zespół: 3 osoby | Narzędzie: **KNIME Analytics Platform**

---

## 📋 Opis projektu

Aplikacja agentowa w KNIME, która symuluje **asystenta firmy sprzątającej**.  
Agent rozmawia z klientem, zbiera informacje o jego domu i przewiduje **cenę sprzątania w EUR** na podstawie wytrenowanego modelu ML.

**Przykładowa rozmowa z agentem:**
> 🧑 *"Dzień dobry, chciałem zamówić sprzątanie"*  
> 🤖 *"Ile m² ma Twoje mieszkanie? Czy masz zwierzęta? Jak często chcesz sprzątanie?"*  
> 🤖 *"Szacowana cena to 67 EUR. Pewność modelu: wysoka (R²=0.91)"*

---

## 📁 Pliki w repozytorium

```
/
├── cleaning_dataset_eur.csv     # Dataset treningowy (1000 wierszy)
├── cleaning_model.pmml          # Zapisany model (po etapie 1)
├── workflows/
│   ├── 01_train_model.knwf      # Trening i zapis modelu
│   ├── tool1_predict.knwf       # Narzędzie 1: predykcja
│   ├── tool2_confidence.knwf    # Narzędzie 2: confidence
│   ├── tool3_text_to_table.knwf # Narzędzie 3: tekst → tabela
│   ├── tool4_feature_importance.knwf  # Narzędzie 4 (bonus)
│   └── agent_main.knwf          # Główny workflow z agentem
└── README.md
```

---

## 🗂️ Dataset

Plik: `cleaning_dataset_eur.csv` | **1000 wierszy**, syntetyczny

| Kolumna | Typ | Opis |
|---|---|---|
| `area_m2` | int | Metraż mieszkania (30–200 m²) |
| `num_rooms` | int | Liczba pokoi (1–8) |
| `num_bathrooms` | int | Liczba łazienek (1–3) |
| `has_pets` | int (0/1) | Czy są zwierzęta domowe |
| `has_garden` | int (0/1) | Czy jest ogródek do sprzątania |
| `cleaning_frequency` | string | Częstotliwość: `weekly` / `biweekly` / `monthly` |
| `cleaning_price_eur` | float | **Cena sprzątania w EUR** (target) |

---

## 👥 Podział pracy

| Etap | Zadanie | Osoba | Status |
|---|---|---|---|
| **Etap 1** | Wczytanie datasetu, One-to-Many, trening modelu, Scorer | Osoba 1 | ✅ ZROBIONE |
| **Etap 2** | Model Writer – zapis modelu do pliku | Osoba 1 | ✅ ZROBIONE |
| **Tool 1** | Workflow: predykcja dla nowych danych (Table Input → Model Reader → Predictor → Table Output) | Osoba 1 | ✅ ZROBIONE |
| **Tool 2** | Workflow: confidence predykcji (przedział ufności ±10% lub R² jako miara) | Osoba 2 | ⏳ DO ZROBIENIA |
| **Tool 3** | Workflow: tekst → tabela przez LLM | Osoba 2 | ⏳ DO ZROBIENIA |
| **Tool 4** | Workflow: feature importance (bonus) | Osoba 3 | ⏳ DO ZROBIENIA |
| **Agent** | Połączenie wszystkich narzędzi w agenta (List Files → Workflow to Tool → Agent Prompter → View Conversation) | Osoba 3 | ⏳ DO ZROBIENIA |

---

## ✅ ETAP 1 – Trening modelu (ZROBIONE)

### Węzły w KNIME (kolejność):
```
CSV Reader → One to Many → Table Partitioner → Linear Regression Learner
                                    ↓                        ↓
                           Regression Predictor ←————————————
                                    ↓
                                 Scorer
```

### Konfiguracja:
- **CSV Reader**: wczytaj `cleaning_dataset_eur.csv`
- **One to Many**: kolumna `cleaning_frequency` → zamienia na `weekly`, `biweekly`, `monthly` (0/1)
- **Table Partitioner**: 80% train (górny port) / 20% test (dolny port), tryb `Random`
- **Linear Regression Learner**: target = `cleaning_price_eur`, includes = wszystkie pozostałe kolumny
- **Regression Predictor**: lewy port ← model z Learnera | prawy port ← dane testowe (dolny z Partitioner)
- **Scorer**: First column = `cleaning_price_eur` | Second column = `Prediction (cleaning_price_eur)`

### Wyniki modelu:
| Metryka | Wartość |
|---|---|
| R² | **0.913** ✅ |
| Mean Absolute Error | 3.683 EUR |
| RMSE | 4.681 EUR |
| MAPE | 6.9% |

---

## ✅ ETAP 2 – Zapis modelu (ZROBIONE)

- Węzeł **Model Writer** podłączony do górnego portu (model) z **Linear Regression Learner**
- Zapisz jako `cleaning_model` w folderze projektu
- Ten plik będzie wczytywany przez Tool 1, 2 i 4

---

## ⏳ TOOL 1 – Predykcja dla nowych danych

> **Cel:** Przyjmuje tabelę z cechami domu → zwraca przewidywaną cenę sprzątania

### Węzły:
```
Container Input (Table) → One to Many → Model Reader → Regression Predictor → Container Output (Table)
```

### Konfiguracja:
- **Container Input (Table)**: nazwij `new_data`
- **One to Many**: tak samo jak w etapie 1 – kolumna `cleaning_frequency`
- **Model Reader**: wskaż plik `cleaning_model` zapisany w Etapie 2
- **Regression Predictor**: podłącz model (lewy) i dane (prawy)
- **Container Output (Table)**: zwraca tabelę z kolumną `Prediction (cleaning_price_eur)`

---

## ⏳ TOOL 2 – Confidence predykcji

> **Cel:** Zwraca pewność/przedział ufności predykcji

### Węzły:
```
Container Input (Table) → One to Many → Model Reader → Regression Predictor → Math Formula → Container Output (Table)
```

### Konfiguracja Math Formula (dodaj 2 kolumny):
- `confidence_low` = `$Prediction (cleaning_price_eur)$ * 0.90`
- `confidence_high` = `$Prediction (cleaning_price_eur)$ * 1.10`

Możesz też dodać kolumnę `r2_score` z wartością stałą `0.913` jako ogólną miarę jakości modelu.

---

## ⏳ TOOL 3 – Tekst → Tabela przez LLM ⭐ KLUCZOWE

> **Cel:** Klient opisuje dom słownie → LLM konwertuje to do tabeli → tabela idzie do Tool 1

### Węzły:
```
Container Input (Text) → LLM Prompter → String to JSON → JSON to Table → Container Output (Table)
```

### Konfiguracja LLM Prompter:
W polu **System Prompt** wpisz:
```
You are a data extraction assistant. Extract home cleaning information from the user's text and return ONLY a valid JSON object with these exact fields:
- area_m2 (integer)
- num_rooms (integer)  
- num_bathrooms (integer)
- has_pets (0 or 1)
- has_garden (0 or 1)
- cleaning_frequency (must be exactly: "weekly", "biweekly", or "monthly")

Return ONLY the JSON, no explanation, no markdown, no backticks.
```

W polu **User Prompt** wpisz:
```
Extract cleaning service information from this text: {{input}}
```

### Przykład działania:
- **Wejście:** *"Mam mieszkanie 75m2, dwa pokoje, jedna łazienka, mam kota, bez ogródka, chcę sprzątanie co dwa tygodnie"*
- **Wyjście JSON:** `{"area_m2": 75, "num_rooms": 2, "num_bathrooms": 1, "has_pets": 1, "has_garden": 0, "cleaning_frequency": "biweekly"}`
- **Wyjście tabela:** gotowa do podania do Tool 1

---

## ⏳ TOOL 4 – Feature Importance (bonus)

> **Cel:** Pokazuje które cechy domu najbardziej wpływają na cenę sprzątania

### Węzły:
```
Model Reader → Linear Regression Inspector → Container Output (Table)
```

### Konfiguracja:
- **Model Reader**: wczytaj `cleaning_model`
- **Linear Regression Inspector**: wyświetla współczynniki regresji dla każdej cechy
- **Container Output**: zwraca tabelę z kolumnami `Feature` i `Coefficient`

Wynik pokaże np. że `area_m2` ma największy wpływ, a `has_pets` mniejszy.

---

## ⏳ AGENT – Połączenie wszystkiego

> **Cel:** Agent który rozmawia z użytkownikiem i używa narzędzi automatycznie

### Węzły:
```
List Files/Folders → Workflow to Tool (x4) → Agent Prompter → View Agent Conversation
```

### Konfiguracja:
1. **List Files/Folders**: wskaż folder z zapisanymi workflow tool1–tool4
2. **Workflow to Tool** × 4: dla każdego narzędzia osobny węzeł, wypełnij:
   - *Name*: np. `predict_price`
   - *Description*: opisz co robi (agent czyta ten opis żeby wiedzieć kiedy użyć narzędzia!)
3. **Agent Prompter** – System prompt:
```
You are a friendly cleaning service assistant. Your job is to help customers estimate the cost of cleaning their home.

Start by greeting the customer. Then ask them (one question at a time) about:
1. The size of their home in square meters
2. Number of rooms
3. Number of bathrooms
4. Whether they have pets (yes/no)
5. Whether they have a garden (yes/no)
6. Preferred cleaning frequency (weekly, every two weeks, or monthly)

Once you have all information, use the text_to_table tool to convert their description, then use predict_price to estimate the cost. Present the result clearly in EUR. Also mention which features affect the price most using the feature_importance tool.

Always respond in the same language the customer uses.
```
4. **View Agent Conversation**: tutaj testujesz rozmowę

---

## 📊 Architektura całego systemu

```
Klient (tekst)
      ↓
[Tool 3: text → tabela]
      ↓
[Tool 1: predykcja ceny]  ←→  [Model: cleaning_model.pmml]
      ↓
[Tool 2: confidence]
      ↓
[Tool 4: feature importance]
      ↓
Agent odpowiada klientowi
```

---

## 🔧 Wymagania techniczne

- KNIME Analytics Platform (najnowsza wersja)
- Rozszerzenia KNIME: **KNIME AI Extension** (do węzłów LLM i Agent)
- Dostęp do modelu LLM przez IAEdu lub inny provider skonfigurowany na zajęciach

---

## 📝 Notatki

- Dataset jest **syntetyczny** – wygenerowany ze wzoru liniowego, stąd wysoki R²=0.913
- Kolumna `cleaning_frequency` **musi** być zakodowana przez One-to-Many przed każdym użyciem modelu
- Przy Tool 3 upewnij się że JSON zwrócony przez LLM ma dokładnie te same nazwy kolumn co dataset