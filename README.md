# aws-serverless-scalable-api

Projet personnel pour s'exercer aux compétences d'**architecte cloud AWS** : conception d'une architecture serverless scalable, gestion des accès IAM en least-privilege, et mise en place d'un pipeline CI/CD sécurisé avec approbation manuelle avant la prod.

> Le focus du projet est l'**architecture et le déploiement**, pas la logique métier. L'API elle-même (gestion de tâches/projets) sert de support simple pour exercer plusieurs patterns d'accès DynamoDB et plusieurs microservices Lambda.

---

## Architecture

| Service | Rôle |
|---|---|
| **Cognito** | Authentification des utilisateurs (User Pool) |
| **API Gateway** | Exposition des microservices, routage des requêtes |
| **Lambda** | Fonctions métier (une par microservice logique) |
| **DynamoDB** | Base de données NoSQL |
| **IAM** | Gestion des droits, rôles d'exécution Lambda et rôles de déploiement |
| **CloudWatch** | Logs et observabilité |

Le tout est décrit en Infrastructure as Code via un **template SAM** (`template.yaml`), déployé automatiquement par une pipeline **GitHub Actions**.

---

## Stratégie multi-environnements (dev / prod)

Le projet ne dispose pas de comptes AWS séparés pour dev et prod : les deux environnements cohabitent **dans le même compte AWS**, isolés logiquement plutôt que physiquement.

- **Un stack CloudFormation par environnement** : `aws-serverless-scalable-api-dev` et `aws-serverless-scalable-api-prod`
- **Toutes les ressources nommées avec un préfixe d'environnement** via un paramètre SAM `Environment` (`dev`/`prod`), ex : `aws-serverless-scalable-api-dev-tasks`
- **Le nom de la ressource devient la frontière de sécurité** : les policies IAM sont scopées sur ce préfixe plutôt que sur un compte AWS dédié

Limite assumée : pas d'isolation de facturation/quotas, et une erreur de scoping IAM pourrait faire fuiter un accès dev vers une ressource prod. Une évolution naturelle serait de passer en multi-compte via AWS Organizations.

---

## Pipeline CI/CD (GitHub Actions)

Le workflow (`.github/workflows/deploy_sam_branch.yaml`) enchaîne :

1. **Lint** — `ruff` + `black --check`
2. **Test** — `pytest`
3. **Build** — `sam build`
4. **Deploy dev** — automatique sur push, via l'environnement GitHub `dev`
5. **Deploy prod** — même pipeline, mais bloqué par une **approbation manuelle** (Required reviewers configurés sur l'environnement GitHub `production`)

### Authentification AWS : OIDC (pas de clés statiques)

Le déploiement utilise l'authentification **OIDC** entre GitHub Actions et AWS (`aws-actions/configure-aws-credentials`), plutôt que des clés IAM stockées en secret :

- Un **fournisseur OIDC** (`token.actions.githubusercontent.com`) est enregistré une fois dans le compte AWS
- Chaque environnement (dev/prod) a son **propre rôle IAM** (`GithubActionsDeployRole-Dev` / `-Prod`), avec :
  - une **trust policy** scopée au repo + à l'environnement GitHub
  - une **permissions policy** least-privilege scopée aux ressources préfixées de cet environnement

---

## Trust policy — point d'attention important

⚠️ **Piège rencontré et documenté ici pour ne pas le reproduire** :

Quand un job GitHub Actions référence un `environment:` (ex. `environment: { name: dev }`), le claim `sub` du token OIDC **change de format**. Il ne suit plus le format classique basé sur la branche :

```
repo:<owner>/<repo>:ref:refs/heads/<branch>
```

mais devient :

```
repo:<owner>@<owner_id>/<repo>@<repo_id>:environment:<environment_name>
```

Exemple réel pour ce projet :
```
repo:Henokagb@72027682/aws-serverless-scalable-api@1355223006:environment:dev
```

Une trust policy écrite avec le format "classique" (même avec un wildcard `*` généreux sur le nom du repo) ne matchera **jamais** ce `sub`, car le préfixe littéral (`owner@id/repo@id`) est différent — d'où l'erreur `Not authorized to perform sts:AssumeRoleWithWebIdentity` malgré une configuration IAM par ailleurs correcte (provider OIDC, SCP, RCP, permissions boundary tous valides).

**Comment vérifier le vrai `sub` envoyé par un run** : ajouter temporairement une étape de debug dans le workflow qui décode le JWT reçu et affiche ses claims (`sub`, `aud`, `ref`, `environment`) — plus fiable que de deviner le format.

### Trust policy — rôle dev (exemple)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": [
        "sts:AssumeRoleWithWebIdentity",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:<OWNER>@<OWNER_ID>/<REPO>@<REPO_ID>:environment:dev"
          ]
        }
      }
    }
  ]
}
```

> `sts:TagSession` est nécessaire car `aws-actions/configure-aws-credentials` attache des tags de session (repo, branche, workflow) à l'appel `AssumeRoleWithWebIdentity` par défaut. Sans cette permission, l'assume role échoue avec le même message générique `Not authorized`.

---

## Permissions policy — principe de scoping

Chaque rôle de déploiement (dev/prod) a une policy de permissions qui autorise uniquement les actions nécessaires à `sam deploy`, scopées par préfixe de nom de ressource :

- `cloudformation:*` sur le stack `aws-serverless-scalable-api-<env>`
- `lambda:*` sur `function:aws-serverless-scalable-api-<env>-*`
- `dynamodb:*` sur `table/aws-serverless-scalable-api-<env>-*`
- `cognito-idp:*` sur les user pools du projet
- `logs:*` sur `log-group:/aws/lambda/aws-serverless-scalable-api-<env>-*`
- `iam:*Role*` + `iam:PassRole` sur `role/aws-serverless-scalable-api-<env>-*` (pour le rôle d'exécution Lambda auto-généré par SAM)
- `s3:*` sur le bucket managé par SAM CLI (`aws-sam-cli-managed-*`)
- `apigateway:*` (non scopable finement par nom de ressource dans IAM)

Ce scoping garantit que le rôle GitHub Actions ne peut toucher **que** les ressources de ce projet et de cet environnement — pas le reste du compte AWS.

---

## Checklist de mise en place

- [ ] Créer le fournisseur OIDC `token.actions.githubusercontent.com` dans IAM (audience `sts.amazonaws.com`)
- [ ] Créer `GithubActionsDeployRole-Dev` avec sa trust policy (format `environment:dev`)
- [ ] Créer `GithubActionsDeployRole-Prod` avec sa trust policy (format `environment:production` ou équivalent)
- [ ] Attacher les permissions policies least-privilege correspondantes à chaque rôle
- [ ] Configurer les **Environments** GitHub (`Settings > Environments`) : `dev` sans protection, `production` avec Required reviewers
- [ ] Ajouter les secrets GitHub `AWS_DEPLOY_ROLE_DEV`, `AWS_DEPLOY_ROLE_PROD` (ARNs des rôles) et la variable `AWS_REGION`
- [ ] Vérifier que tous les noms de ressources dans `template.yaml` respectent le préfixe `aws-serverless-scalable-api-<env>-`
