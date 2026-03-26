# Clips Fortnite — Contexte projet

## Ce que fait ce projet

Pipeline automatisé de détection de highlights dans des VODs Twitch Fortnite.
Objectif final : produire des clips directement utilisables par Silixio (monteur) pour la chaîne YouTube "Dentoz - Fortnite Best of" (~200k abonnés).

**Un clip "utilisable" = un moment spectaculaire (gros kill, clutch, fail drôle, réaction forte, Victory Royale) avec un timing précis, prêt à être intégré dans un best-of sans devoir chercher le moment fort dans la vidéo.**

## Architecture du pipeline

```
streamers.txt / KNOWN_FR_STREAMERS / auto-discover (Twitch Helix)
        ↓
   downloader.py — Télécharge VODs + chat (TwitchDownloaderCLI)
        ↓
   main.py — Orchestre les 5-6 analyses en parallèle :
        ├── analyzers/audio.py      — Silero VAD + volume spike detection
        ├── analyzers/whisper_analyzer.py — faster-whisper (GPU) + keyword scoring
        ├── analyzers/chat_analyzer.py   — Pics d'activité chat Twitch (volume)
        ├── analyzers/chat_content_analyzer.py — Analyse qualitative chat via Ollama (Sprint 15)
        ├── analyzers/video_analyzer.py  — Victory Royale (couleurs) + kill feed OCR (Sprint 19)
        ├── analyzers/killfeed_ocr.py    — Sprint 19 : EasyOCR sur zone killfeed (remplace diff pixels)
        ├── analyzers/ai_analyzer.py     — Re-ranking Ollama/Mistral + transcript complet casters (Sprint 17)
        └── analyzers/fail_detector.py   — Sprint 23 : Détection fails drôles (audio drop + emotes LUL)
        ↓
   scoring/engine.py — Normalisation percentile + fusion multi-source + scoring pondéré + fallback Ollama
        ↓
   scoring/sequences.py — Sprint 13 : Fusion highlights proches en séquences narratives
        ↓
   scoring/ml_ranker.py — Sprint 23 : Scoring ML LightGBM (optionnel, remplace heuristique si entraîné)
        ↓
   clipper/extractor.py — Découpe FFmpeg (copy, pas de ré-encodage) avec offset par source
        ↓
   chat/renderer.py — Sprint 18 : Vidéo chat Twitch séparée par clip (TwitchDownloaderCLI)
        ↓
   review.py — Page HTML de review + rapport JSON
        ↓
   delivery.py — Sprint 14 : Upload Google Drive (clips + report + Sheet résumé + feedback)
        ↓
   feedback.py + auto_tune.py — Sprint 16/22 : Boucle de feedback + auto-tune 3 params
        ↓
   metrics.py — Sprint 22 : Precision@K + historique métriques
        ↓
   youtube/ — Sprint 12 : Intégration YouTube
        ├── analyzer.py             — Parse CSV YouTube Studio + scoring vidéos
        ├── title_patterns.py       — Patterns titres top 20% vs bottom 20%
        ├── suggest.py              — Titre suggéré + candidats miniature + potentiel YT
        └── streamer_performance.py — Ranking streamers par "valeur YouTube"
```

## Stack technique

- **Python 3.12** sur Windows
- **FFmpeg** pour extraction audio et découpe vidéo
- **faster-whisper** (GPU CUDA, modèle "small") pour la transcription
- **Silero VAD** (PyTorch) pour la détection vocale
- **OpenCV + NumPy** pour l'analyse vidéo
- **Ollama + Mistral** (local) pour le re-ranking IA
- **TwitchDownloaderCLI** pour le téléchargement VODs/chat
- **API Helix Twitch** (OAuth client_credentials) pour la découverte de streamers
- **EasyOCR** (GPU) pour la détection killfeed par OCR (Sprint 19)
- **Google Drive API** (service account) pour la livraison des clips à Silixio

## Fichiers de config importants

- `config.py` — TOUS les paramètres ajustables (seuils, poids, durées, chemins)
- `streamers.txt` — Liste manuelle de streamers à suivre (jolavanille, nikof)
- `known_streamers.json` — Cache auto-découverte
- `processed_vods.json` — Historique des VODs déjà traitées
- `SPEC.md` — Plan d'amélioration détaillé par sprints
- `service_account.json` — Credentials Google (ne pas committer)
- `config_overrides.json` — Overrides auto-tune (Sprint 16, généré automatiquement)
- `logs/` — Logs quotidiens (Sprint 14)

## Conventions de code

- Docstrings en français, commentaires explicatifs dans le code
- Logs via `utils/logger.py` (module `log`)
- Config centralisée dans `config.py` (jamais de magic numbers dans le code)
- Dataclasses pour les structures de données (AudioPeak, WhisperHighlight, etc.)
- Les analyseurs retournent des listes triées par score décroissant
- Commentaires de sprint (`# Sprint N : ...`) pour tracer les changements

## Modes d'exécution

```bash
python main.py <vod.mp4>          # Traite une VOD
python main.py --batch <dossier>  # Traite toutes les VODs d'un dossier
python main.py --auto             # Download + analyse + cleanup (tâche planifiée)
python main.py --discover         # Scan streamers FR via Helix → cache
python gui.py                     # Interface graphique (Tkinter)
```

## État des améliorations (sprints)

Voir `SPEC.md` pour les détails de chaque sprint.

| Sprint | Statut | Résumé |
|--------|--------|--------|
| 1 — Scoring | ✅ FAIT | Normalisation percentile, cap single-source ×0.4, poids recalibrés |
| 2 — Whisper | ✅ FAIT | COMMENTARY_PATTERNS supprimés, exclamations durcies, seuil 0.6 |
| 3 — Timing | ✅ FAIT | Buffer -10s/+20s, max 45s, distance 120s, offset par source |
| 4 — Vidéo | ✅ FAIT | Seuils killfeed durcis, max 10 détections, score plafonné 0.6 |
| 5 — VR fix | ✅ FAIT | Seuil doré 0.25, luminosité 0.6, min 4 frames consécutives |
| 6 — Validation | ✅ FAIT | Review HTML + boutons feedback + export JSON + validate.py |
| 7 — Streamers | ✅ FAIT | API Helix officielle, liste KNOWN_FR_STREAMERS, tri par followers, cache 30j |
| 8 — Pré-filtre | ✅ FAIT | Scan audio rapide (RMS), skip VODs calmes/AFK, prefilter.py |
| 9 — Parallélisme | ✅ FAIT | Pipeline 3 étages dans run_auto, AI_MAX_CANDIDATES réduit à 15 |
| 10 — Profils | ✅ FAIT | Profils caster/gameplay_solo, auto-détection VAD, poids par profil |
| 11 — IA profil | ✅ FAIT | Prompt Ollama adapté au profil (caster vs gameplay_solo) |
| 12 — YouTube | ✅ FAIT | Analyse perf YouTube, patterns titres, suggestions, ranking streamers |
| 13 — Séquences | ✅ FAIT | Fusion highlights proches en séquences narratives, contexte IA élargi |
| 14 — Auto + Drive | ✅ FAIT | Health checks, logging fichier, upload Google Drive, cleanup VOD auto |
| 15 — Chat content | ✅ FAIT | Analyse qualitative chat via Ollama, 6ème source scoring, profil caster |
| 16 — Feedback loop | ✅ FAIT | Boucle de feedback, auto-tune seuils, rapport hebdo, config_overrides.json |
| 17 — Optimisation vitesse | ✅ FAIT | Priorité streamers, Whisper adaptatif, killfeed off, transcript caster, retry download |
| 18 — Chat clips séparés | ✅ FAIT | Vidéo chat Twitch séparée par clip, upload Drive, review HTML |
| 19 — OCR killfeed | ✅ FAIT | EasyOCR remplace diff pixels, détection streaks, zone haut-droite |
| 20 — Scoring v2 | ✅ FAIT | Single-source 0.65, poids rééquilibrés (IA -10%, vidéo +5%), OCR 0.55, fallback Ollama |
| 21 — Détection v2 | ✅ FAIT | Buffers profil, emotes hype, vocabulaire streamer, regex souples, chat spike 2.0 |
| 22 — Métriques | ✅ FAIT | Precision@K auto, auto-tune 3 params, VODs de référence annotées |
| 23 — ML + fails | ✅ FAIT | LightGBM scoring, fail detector, offsets chat adaptatifs |

## Problèmes connus

1. ~~**Victory Royale : 44 faux positifs**~~ — Corrigé Sprint 5.
2. ~~**Découverte streamers limitée**~~ — Corrigé Sprint 7.
3. ~~**Vitesse insuffisante**~~ — Corrigé Sprints 8-9.
4. ~~**Pas de profil par streamer**~~ — Corrigé Sprint 10.
5. ~~**Killfeed ~4000 faux positifs (diff pixels)**~~ — Corrigé Sprint 19 (remplacé par OCR).

## Définition de "terminé" pour un clip

Un clip est UTILISABLE si :
- [ ] Il contient un moment clairement identifiable (kill, clutch, fail, réaction, VR)
- [ ] Le moment fort est dans les 15 premières secondes du clip
- [ ] Le son est exploitable (streamer réagit, chat réagit, ou action parle d'elle-même)
- [ ] Il dure 20-45 secondes (pas 60s de filler)
- [ ] Silixio peut l'intégrer dans un best-of sans chercher où est le moment fort

## Commandes utiles

```bash
# Tester sur une seule VOD
python main.py "chemin/vers/vod.mp4"

# Vérifier qu'Ollama est prêt
curl http://localhost:11434/api/tags

# Lancer le mode automatique complet
python main.py --auto
```
