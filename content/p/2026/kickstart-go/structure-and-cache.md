+++
date = '2026-01-02T15:20:57-06:00'
title = 'Estructura y cache'
author = 'Gustavo Calvo'
tags = ['dev', 'golang']
+++

Creamos un proyecto con la siguiente estructura:

```
app
├── auth          # Auth providers
├── cache         # Key-value store
├── config        # Configuration
├── contrib       # Extra
├── database      # RDB
├── i18n          # Internationalization
├── middleware    # Logging, CORS, context timeout
├── sessions      # Session management
└── storage       # Object storage
```

Ahora, inicializamos como módulo de Go:

```sh
go mod init app

# Opcionalmente:
go mod init github.com/user/repo
```

Finalmente, inicializamos el repositorio localmente

```sh
git init
git add -A
git commit -m init
```

---

# Módulo `cache`

`cache/internal.go`
```go {title="cache/internal.go"}
package cache

type errStr string

func (e errStr) Error() string {
	return string(e)
}
```

`cache/cache.go`
```go {title="cache/cache.go"}
// Package cache implements key-value-like interface 'Cache'
package cache

import (
	"context"
)

const (
	ErrNotFound = errStr("key not found")
)

type Cache interface {
	Set(ctx context.Context, key, value string) (err error)
	Get(ctx context.Context, key string) (value string, err error)
	Del(ctx context.Context, key string) (err error)
	GetAll(ctx context.Context) (values map[string]string, err error)
}
```

`cache/memory.go`
```go {title="cache/memory.go"}
package cache

import (
	"context"
	"sync"
)

type MemoryStore struct {
	m sync.Map // map[string]string
}

var _ Cache = (*MemoryStore)(nil)

func NewMemoryStore() *MemoryStore {
	return &MemoryStore{m: sync.Map{}}
}

func (s *MemoryStore) Set(_ context.Context, key, value string) error {
	s.m.Store(key, value)
	return nil
}

func (s *MemoryStore) Get(_ context.Context, key string) (string, error) {
	v, ok := s.m.Load(key)
	if !ok {
		return "", ErrNotFound
	}
	return v.(string), nil
}

func (s *MemoryStore) Del(_ context.Context, key string) error {
	s.m.Delete(key)
	return nil
}

func (s *MemoryStore) GetAll(_ context.Context) (map[string]string, error) {
	values := make(map[string]string)

	s.m.Range(func(k, v any) bool {
		values[k.(string)] = v.(string)
		return true
	})

	return values, nil
}
```

`cache/redis.go`
```go {title="cache/redis.go"}
package cache

import (
	"context"

	"github.com/redis/go-redis/v9"
)

type RedisStore struct {
	r *redis.Client
}

var _ Cache = (*RedisStore)(nil)

func NewRedisStore(opt *redis.Options) *RedisStore {
	return &RedisStore{r: redis.NewClient(opt)}
}

func (s *RedisStore) Set(ctx context.Context, key, value string) error {
	return s.r.Set(ctx, key, value, 0).Err()
}

func (s *RedisStore) Get(ctx context.Context, key string) (string, error) {
	val, err := s.r.Get(ctx, key).Result()
	if err != nil {
		if err == redis.Nil {
			return "", ErrNotFound
		}
		return "", err
	}
	return val, nil
}

func (s *RedisStore) Del(ctx context.Context, key string) error {
	return s.r.Del(ctx, key).Err()
}

func (s *RedisStore) GetAll(ctx context.Context) (map[string]string, error) {
	values := make(map[string]string)

	var cursor uint64
	for {
		keys, nextCursor, err := s.r.Scan(ctx, cursor, "*", 100).Result()
		if err != nil {
			return nil, err
		}

		if len(keys) > 0 {
			vals, err := s.r.MGet(ctx, keys...).Result()
			if err != nil {
				return nil, err
			}

			for i, v := range vals {
				if v == nil {
					continue
				}
				values[keys[i]] = v.(string)
			}
		}

		cursor = nextCursor
		if cursor == 0 {
			break
		}
	}

	return values, nil
}
```
