# Azure Monitor Dashboard

Dashboard do monitorowania Azure Functions, Web API i ADF z danymi z Application Insights.
Klucze API przechowywane wyłącznie w `localStorage` przeglądarki — bezpieczne w publicznym repo.

## Struktura repozytorium

```
├── index.html                        # Dashboard (jedyny plik aplikacji)
├── GitVersion.yml                    # Konfiguracja wersjonowania semantycznego
├── README.md
└── .github/
    └── workflows/
        └── deploy.yml                # GitHub Actions — GitVersion + deploy na Pages
```

## Pierwsze uruchomienie (5 minut)

### 1. Utwórz repozytorium i wgraj pliki

```bash
git init azure-monitor-dashboard
cd azure-monitor-dashboard

# Skopiuj index.html, GitVersion.yml, README.md i .github/ do tego folderu
git add .
git commit -m "Initial dashboard"
git remote add origin https://github.com/TWOJA_NAZWA/azure-monitor-dashboard.git
git push -u origin main
```

### 2. Włącz GitHub Pages

```
Settings → Pages → Source → GitHub Actions
```

Po pierwszym push workflow uruchomi się automatycznie.
URL: `https://TWOJA_NAZWA.github.io/azure-monitor-dashboard`

### 3. Skonfiguruj klucze Application Insights

Klucze nie są przechowywane w repo — każdy użytkownik ustawia je sam w przeglądarce:
1. Otwórz dashboard → kliknij **🔑 Skonfiguruj klucze**
2. Dla każdego środowiska podaj Application ID i API Key
   (Azure Portal → App Insights → Configure → API Access)

---

## Wersjonowanie z GitVersion

Projekt używa [GitVersion](https://gitversion.net) do automatycznego obliczania wersji
według [Semantic Versioning 2.0](https://semver.org). Wersja jest wstrzykiwana do `index.html`
podczas buildu (zastąpienie placeholderów `__VERSION__` itp.).

### Jak działa

```
Tag v1.0.0
    │
    ├─ push na main          → 1.0.1        (patch bump)
    ├─ branch feature/xyz    → 1.1.0-alpha.1
    ├─ branch release/1.1    → 1.1.0-rc.1
    ├─ merge release → main  → 1.1.0
    └─ branch hotfix/login   → 1.0.2-hotfix.1
```

### Strategie bump przez commit message

```bash
git commit -m "fix login bug"                    # → patch bump (domyślnie na main)
git commit -m "add new chart +semver: minor"     # → minor bump (1.1.0 → 1.2.0)
git commit -m "breaking API change +semver: major"  # → major bump (1.x → 2.0.0)
git commit -m "update docs +semver: none"        # → brak bumpa
```

### Tworzenie release

```bash
# Opcja A — tag bezpośrednio (najprostsze)
git tag v1.2.0
git push origin v1.2.0

# Opcja B — branch release (GitFlow)
git checkout -b release/1.2
git push origin release/1.2
# → wersja: 1.2.0-rc.1, rc.2, ...
# Po merge do main → 1.2.0
```

### Ręczny deploy z nadpisaną wersją

W GitHub Actions → `Deploy Azure Monitor Dashboard` → `Run workflow`:
- pole `force_version`: wpisz np. `2.0.0`

### Badge wersji na dashboardzie

Po deploymencie w prawym górnym rogu headera pojawia się badge `v1.2.3`
z linkiem do releases. Po najechaniu tooltip pokazuje:
`Build: 1.2.3+Branch.main.Sha.abc1234 · SHA: abc1234 · main`

---

## Aktualizacja dashboardu

```bash
# Edytujesz index.html, następnie:
git add index.html
git commit -m "fix: poprawka wykresu latency +semver: patch"
git push
# → GitVersion oblicza nową wersję → deploy w ~90 sekund
```

## Bezpieczeństwo

| Aspekt | Szczegół |
|---|---|
| Repo | Może być **publiczne** — brak sekretów w kodzie |
| Klucze AI | Tylko w `localStorage` przeglądarki użytkownika |
| Uprawnienia klucza | Ustaw `Read telemetry only` w App Insights |
| GitVersion token | Nie wymaga dodatkowych tokenów — używa domyślnego `GITHUB_TOKEN` |
