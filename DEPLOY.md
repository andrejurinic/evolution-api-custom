# Deploy (GHCR + Coolify)

Este fork publica a própria imagem no GitHub Container Registry e o Coolify puxa
essa imagem. Não há build no VPS.

Imagem: `ghcr.io/andrejurinic/evolution-api-custom` (pacote público)

## Tags publicadas

| Tag | Quem move | Para que serve |
| --- | --- | --- |
| `main` | todo push na `main` | último build da branch; **não** identifica o que está no ar |
| `<versão>` (ex.: `2.3.7_3`) | só o release da tag `v<versão>` | a versão liberada; é o que o Coolify deve apontar |
| `sha-<commit>` | todo build | rastreio e rollback fino, por commit |

`main` é mutável: dois deploys diferentes podem tê-la apontando para imagens
diferentes. Por isso o Coolify fica pinado na tag de versão.

## Liberar uma versão

1. Faça o merge do PR na `main`. O workflow *Publish main image to GHCR* builda e
   publica `main`, a versão atual do `package.json` e `sha-<commit>`.
2. Suba a versão no `package.json` (`2.3.7_3` → `2.3.7_4`) e faça o commit na
   `main`. A convenção do fork é `<versão upstream>_<n>`, porque a rota raiz da
   API devolve `packageJson.version` e é isso que aparece no rodapé do manager.
3. Crie a tag e empurre:

   ```bash
   git checkout main && git pull
   git tag v2.3.7_4
   git push origin v2.3.7_4
   ```

4. O workflow *Release* confere se a tag bate com o `package.json`, publica a
   imagem com a tag da versão e abre o GitHub Release com a referência da imagem
   e o digest.

## Colocar no ar

No Coolify, o recurso aponta para `ghcr.io/andrejurinic/evolution-api-custom:<versão>`:

1. troque a tag da imagem para a versão nova;
2. **Deploy**.

O container roda as migrations sozinho no start (`ENTRYPOINT` →
`Docker/scripts/deploy_database.sh` → `npm run db:deploy`), então não há passo
manual de banco. Se as migrations falharem, o container sai com erro em vez de
subir com o schema errado.

Alternativa sem abrir o painel: o workflow *Deploy to Coolify* (manual, em
Actions) chama o webhook de deploy. Ele precisa de dois secrets no repositório:

- `COOLIFY_WEBHOOK_URL` — Deploy Webhook URL do recurso, copiada do Coolify;
- `COOLIFY_API_TOKEN` — token de API do Coolify com permissão de deploy.

Esse webhook manda o Coolify redeployar **a configuração que já está lá**: ele
não troca a tag da imagem. Para subir uma versão nova por esse caminho, a tag
precisa ter sido alterada antes no painel, ou o recurso precisa estar apontando
para `main`.

## Rollback

Aponte o recurso para a tag da versão anterior (ou para o `sha-<commit>`
correspondente) e faça o deploy. As duas tags são imutáveis, então a imagem que
volta é exatamente a que estava no ar.

Rollback de migration não é automático: se a versão que sai tiver criado
migrations, verifique o schema antes de voltar.

## Conferir o que está no ar

```bash
curl -s https://evo.psibot.com.br/ | head
```

A rota raiz devolve a `version` do `package.json` da imagem em execução.
