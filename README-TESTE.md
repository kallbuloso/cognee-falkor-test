# Cognee + FalkorDB — Stack paralela de homologação

Esta Stack usa:

- Cognee API customizada com o adapter `cognee-community-hybrid-adapter-falkor==0.3.1`;
- FalkorDB para grafo e vetores;
- PostgreSQL apenas para dados relacionais do Cognee;
- EBAC e autenticação ativados;
- volumes e portas diferentes da Stack Neo4j atual.

## 1. Preparar o arquivo de ambiente

```bash
cp .env.example .env
nano .env
```

Para gerar secrets no WSL:

```bash
openssl rand -hex 32
```

Use um valor diferente para cada secret.

## 2. Construir a imagem Cognee com o adapter

```bash
docker build \
  -f Dockerfile.cognee-falkor \
  -t cognee-falkor:main-5b32da7 \
  .
```

## 3. Confirmar o registro do adapter

```bash
docker run --rm \
  --entrypoint /app/.venv/bin/python \
  cognee-falkor:main-5b32da7 \
  -c "from cognee.infrastructure.databases.graph.supported_databases import supported_databases as g; from cognee.infrastructure.databases.vector.supported_databases import supported_databases as v; print('graph=', sorted(g)); print('vector=', sorted(v))"
```

As duas listas devem conter `falkor`.

## 4. Implantar no Swarm local

```bash
docker stack deploy \
  --resolve-image never \
  -c compose.falkor.yaml \
  cogneefalkor
```

## 5. Acompanhar os serviços

```bash
docker stack services cogneefalkor
```

```bash
docker service logs -f cogneefalkor_cognee
```

```bash
docker service logs -f cogneefalkor_falkordb
```

## 6. Testar os endpoints

```bash
curl -i http://localhost:8030/health
```

Swagger:

```text
http://localhost:8030/docs
```

FalkorDB Browser:

```text
http://localhost:3001
```

## 7. Login

```bash
curl -s -X POST http://localhost:8030/api/v1/auth/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "username=admin@localhost.test" \
  --data-urlencode "password=SUA_SENHA"
```

A resposta deve conter `access_token`.

Caso o usuário inicial não seja criado automaticamente pela imagem usada, registre:

```bash
curl -i -X POST http://localhost:8030/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@localhost.test","password":"SUA_SENHA"}'
```

Depois repita o login.

## Observações

1. Não reutilize os volumes da Stack Neo4j.
2. Não publique a porta 6379.
3. O Browser em 3001 é apenas para homologação local.
4. O MCP precisará de `COGNEE_API_TOKEN` para executar operações autenticadas.
5. Não use esta configuração sem senha do FalkorDB na VPS.
