# EWC 2026 Dota Fantasy DB Guide

База: `D:\test\test\ewc_2026_fantasy_compact.sqlite`

Главный ноутбук: `D:\test\test\02_ewc2026_fact_agent_FINAL_clean.ipynb`

## Как пользоваться

Для обычного анализа используй публичные views с префиксом `analytics_`. Они специально сделаны как чистый верхний слой, чтобы не лазить по внутренним таблицам.

Главные views:

- `analytics_player_maps`: fantasy score каждого игрока на каждой карте по текущему default-профилю.
- `analytics_team_role_maps`: одна строка на команду-карту: `avg_core_fantasy_score`, `mid_fantasy_score`, `avg_support_fantasy_score`.
- `analytics_reliable_players`: reliability-v2 игроков с `low_estimate`, `expected_estimate`, `high_estimate`, `confidence_label`.
- `analytics_reliable_role_slots`: reliability-v2 для `core_pair`, `mid_single`, `support_pair`.
- `analytics_optimizer_players`: optimizer привлекательности игроков под текущий banner profile.
- `analytics_optimizer_role_slots`: optimizer привлекательности role-slots.
- `analytics_rosters`: официальные ники и позиции из Liquipedia.
- `analytics_ti2026_teams`: команды TI 2026, qualification path и флаг `has_ewc_player_data`.
- `analytics_sources`: source cache и provenance внешних источников.
- `analytics_scoring_formula`: текущая fantasy-формула и banner multipliers.
- `analytics_reliability_backtest`: backtest reliability-v2.
- `analytics_db_objects`: краткий каталог рекомендуемых объектов базы.

## Core Tables

Внутренние таблицы нужны для пересчета и аудита:

- `matches`: 157 карт EWC 2026, включая `stage_name` и `stage_bucket`.
- `player_game_fantasy_summary`: базовые player-map stats и battlepass points.
- `fantasy_player_map_stat_points`: разложение очков по категориям статистики.
- `fantasy_player_map_scores`: player-map fantasy score для каждого scoring profile.
- `fantasy_team_role_map_scores`: role-category fantasy score для каждого scoring profile.
- `fantasy_scoring_profiles`, `fantasy_scoring_profile_stats`, `fantasy_scoring_profile_banners`: профили подсчета fantasy.
- `fantasy_reliability_v2_player_predictions`, `fantasy_reliability_v2_role_slot_predictions`: reliability-v2 model layer.
- `fantasy_banner_optimizer_recommendations`: optimizer recommendations.
- `player_identity_registry`, `liquipedia_team_rosters`: official identity/positions.
- `ti_qualified_teams`, `external_source_cache`, `dota_heroes`: внешние источники и справочники.

## Частые SQL

Топ fantasy-карт pos1 среди TI-qualified:

```sql
SELECT fantasy_score, official_name, team_name, hero_name, match_id,
       qualification_path, ti_region
FROM analytics_player_maps
WHERE official_position = 1
  AND ti2026_qualified = 1
ORDER BY fantasy_score DESC
LIMIT 15;
```

Надежные игроки pos1:

```sql
SELECT reliability_score_1_100, official_name, team_name, predicted_score_raw,
       low_estimate, expected_estimate, high_estimate, confidence_label
FROM analytics_reliable_players
WHERE official_position = 1
  AND recommended_default = 1
ORDER BY reliability_score_1_100 DESC
LIMIT 15;
```

Optimizer для TI-qualified pos1:

```sql
SELECT optimizer_score_1_100, official_name, team_name, predicted_score_raw,
       best2_series_score, repeatability_ratio, spike_gap
FROM analytics_optimizer_players
WHERE optimizer_scope = 'ti2026'
  AND official_position = 1
ORDER BY optimizer_score_1_100 DESC
LIMIT 15;
```

Сводка по ролям на каждой карте:

```sql
SELECT match_date, stage_name, team_name, opponent_name,
       avg_core_fantasy_score, mid_fantasy_score,
       avg_support_fantasy_score, team_role_fantasy_score
FROM analytics_team_role_maps
ORDER BY match_date, team_name
LIMIT 30;
```

## Fantasy Score

Итог карты:

```text
fantasy_score = base_points_total + profile_bonus_points
```

`base_points_total` считается из battlepass-категорий. `profile_bonus_points` добавляет выбранные banner stats по официальной роли игрока.

## Reliability

Reliability-v2 оценивает не среднюю карту, а повторяемый потолок под fantasy-механику:

- цель: лучший матч как сумма двух лучших карт серии;
- признаки: best2-series, second-best, top2/top3, p75, recent form;
- штрафы: одиночный spike и volatility;
- shrinkage: притягивание к медиане роли при малой выборке;
- шкала: 1-100 внутри роли или role-slot.

Интервалы `low_estimate` / `expected_estimate` / `high_estimate` являются эвристическим uncertainty band, а не строгим статистическим доверительным интервалом.

Саппорты сохранены, но исключены из default recommendations, потому что support-статистика в базе неполная.

## Python Helpers

```python
from ewc_fact_agent_tools import ask, explain_sql_plan

print(ask("top 15 fantasy pos1 players from TI 2026 qualified teams").answer_markdown)
display(explain_sql_plan("top 15 fantasy pos1 players from TI 2026 qualified teams"))
```

Полезные функции:

- `top_fantasy_maps(...)`
- `reliable_players_v2(...)`
- `reliable_role_slots_v2(...)`
- `banner_optimizer_players(...)`
- `banner_optimizer_role_slots(...)`
- `roster(team)`
- `ti_qualified_teams()`
- `source_cache_status()`
- `scoring_formula()`

## Dashboard

```powershell
streamlit run D:\test\test\ewc_fantasy_dashboard.py
```

Если `streamlit` не установлен:

```powershell
pip install -r D:\test\test\requirements_dashboard.txt
```

## Проверка

```powershell
python D:\test\test\regression_tests.py
python D:\test\test\final_project_validation.py
```

## Cleanup

После public analytics cleanup база содержит 37 таблиц и 14 views.

Backup:

```text
D:\test\test\ewc_2026_fantasy_compact.backup_before_public_analytics_layer.sqlite
```

Отчет:

```text
D:\test\test\db_public_analytics_layer_report.md
```

Что убрано:

- старый reliability-v1 слой;
- старые helper/source tables без уникальной информации;
- старые derived `v_*` views, замененные публичными `analytics_*`.
