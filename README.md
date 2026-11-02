# goit-rdb-hw-02

Homework 1 (Topic 2): a data schema for movie recommendations on MovieLens Latest Small, with offline and online feature store layers and training data free of data leakage.

Everything is done in [hw02_movielens_feature_store.ipynb](hw02_movielens_feature_store.ipynb). PostgreSQL 16 starts through `pgserver`, the dataset is downloaded from GroupLens into `data/`, so the notebook passes Restart & Run All both in Colab and locally (`uv sync`).

## Files

- [hw02_movielens_feature_store.ipynb](hw02_movielens_feature_store.ipynb): DDL, loading, historical features, AS-OF queries, checks, normalization notes and reflection.
- [er_diagram.png](er_diagram.png): the ER diagram; it is also embedded in the notebook.
- [er_diagram.mmd](er_diagram.mmd): the same diagram as a text DSL (Mermaid); GitHub renders it below.

## What is inside

- **Entities:** `hw2_users`, `hw2_movies`, `hw2_genres`. Two N:M relationships through bridge tables: `users ↔ movies` through `hw2_ratings`, `movies ↔ genres` through `hw2_movie_genres`. The full dataset is loaded: 610 users, 9,742 movies, 100,836 ratings.
- **Offline:**
  - `hw2_user_features_offline`: 13 feature versions for users 106, 272 and 557 (five for 557) with `[valid_from, valid_to)`.
  - `hw2_training_events`: 13 events with `prediction_ts` and the `liked` label.
- **Online:** `hw2_user_features_online`, one denormalized row per user, refreshed from the current offline version through `ON CONFLICT DO UPDATE`.
- **Checks:**
  - feature values are recomputed from `hw2_ratings`, 0 differences;
  - 0 overlapping intervals and 0 gaps between versions;
  - the AS-OF join returns 13 rows, while a naive join on `user_id` returns 57, 35 of them with future features.

```mermaid
erDiagram
  hw2_users ||--o{ hw2_ratings : "rates (1:N)"
  hw2_movies ||--o{ hw2_ratings : "is rated (1:N)"
  hw2_movies ||--|{ hw2_movie_genres : "has (1:N)"
  hw2_genres ||--o{ hw2_movie_genres : "groups (1:N)"
  hw2_users ||--o{ hw2_user_features_offline : "has versions (1:N)"
  hw2_genres |o--o{ hw2_user_features_offline : "top genre (0..1:N)"
  hw2_users ||--o| hw2_user_features_online : "serves latest (1:0..1)"
  hw2_users ||--o{ hw2_training_events : "is labelled (1:N)"
  hw2_movies ||--o{ hw2_training_events : "target movie (1:N)"

  hw2_users {
    INTEGER user_id PK
    TIMESTAMPTZ first_seen_at "NOT NULL"
    TIMESTAMPTZ last_seen_at "NOT NULL"
  }
  hw2_movies {
    INTEGER movie_id PK
    TEXT title "NOT NULL"
    SMALLINT release_year "CHECK 1870..2100"
    VARCHAR(10) imdb_id
    INTEGER tmdb_id
  }
  hw2_genres {
    VARCHAR(32) genre_name PK
  }
  hw2_movie_genres {
    INTEGER movie_id PK, FK
    VARCHAR(32) genre_name PK, FK
  }
  hw2_ratings {
    INTEGER user_id PK, FK
    INTEGER movie_id PK, FK
    NUMERIC rating "(2,1) CHECK 0.5..5.0 step 0.5"
    TIMESTAMPTZ rated_at "NOT NULL"
  }
  hw2_user_features_offline["OFFLINE hw2_user_features_offline"] {
    INTEGER user_id PK, FK
    TIMESTAMPTZ valid_from PK
    TIMESTAMPTZ valid_to "NULL = current"
    INTEGER ratings_count "CHECK >= 0"
    NUMERIC avg_rating "(3,2) CHECK 0.5..5.0"
    NUMERIC high_rating_share "(4,3) CHECK 0..1"
    VARCHAR(32) top_genre FK
  }
  hw2_training_events["OFFLINE hw2_training_events"] {
    BIGINT event_id PK
    INTEGER user_id FK
    INTEGER movie_id FK
    TIMESTAMPTZ prediction_ts "NOT NULL"
    TIMESTAMPTZ label_observed_at "CHECK >= prediction_ts"
    BOOLEAN liked "NOT NULL, target"
  }
  hw2_user_features_online["ONLINE hw2_user_features_online"] {
    INTEGER user_id PK, FK
    INTEGER ratings_count
    NUMERIC avg_rating "(3,2)"
    NUMERIC high_rating_share "(4,3)"
    VARCHAR(32) top_genre "denormalized copy"
    TIMESTAMPTZ feature_ts "NOT NULL"
    TIMESTAMPTZ updated_at "NOT NULL"
  }

```
