# Skill rules examples

Example configurations covering different trigger types and use cases. For backend/frontend/TDD examples, see the main SKILL.md.

## Technology-specific (file triggers only)

```json
{
  "database-prisma": {
    "type": "technology",
    "priority": "high",
    "promptTriggers": {
      "keywords": ["database", "prisma", "schema", "migration", "query", "orm"],
      "intentPatterns": [
        "(create|update|modify).*?(schema|model|migration)",
        "(query|fetch|get).*?database"
      ]
    },
    "fileTriggers": {
      "pathPatterns": ["**/prisma/**", "**/*.prisma"]
    }
  },
  "devops-docker": {
    "type": "technology",
    "priority": "medium",
    "promptTriggers": {
      "keywords": ["docker", "dockerfile", "container", "deployment", "CI/CD", "kubernetes"],
      "intentPatterns": [
        "(create|build|configure).*?(docker|container)",
        "(deploy|release|publish)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": ["**/Dockerfile", "**/.github/workflows/**", "**/docker-compose.yml"]
    }
  }
}
```

## Security (high-priority, keyword-heavy)

```json
{
  "critical-security": {
    "type": "security",
    "priority": "high",
    "enforcement": "suggest",
    "promptTriggers": {
      "keywords": [
        "security", "vulnerability", "authentication", "authorization",
        "SQL injection", "XSS", "CSRF", "password", "token"
      ],
      "intentPatterns": [
        "secur(e|ity)",
        "(auth|password|token).*(implement|handle|store)"
      ]
    }
  }
}
```

## Multi-repo (path-based routing)

```json
{
  "frontend-mobile": {
    "type": "domain",
    "priority": "high",
    "promptTriggers": {
      "keywords": ["mobile", "ios", "android", "react native"],
      "intentPatterns": ["(create|build).*?(screen|component)"]
    },
    "fileTriggers": {
      "pathPatterns": ["/mobile/**", "/apps/mobile/**"]
    }
  },
  "frontend-web": {
    "type": "domain",
    "priority": "high",
    "promptTriggers": {
      "keywords": ["web", "website", "react", "nextjs"],
      "intentPatterns": ["(create|build).*?(page|component)"]
    },
    "fileTriggers": {
      "pathPatterns": ["/web/**", "/apps/web/**"]
    }
  }
}
```

## Advanced pattern matching

Negative lookahead to exclude false positives:

```json
{
  "backend-dev-guidelines": {
    "promptTriggers": {
      "intentPatterns": [
        "(?!.*test).*backend"
      ]
    }
  }
}
```

Compound patterns for database migrations:

```json
{
  "database-migration": {
    "promptTriggers": {
      "intentPatterns": [
        "(create|generate|run).*(migration|schema change)",
        "(add|remove|modify).*(column|table|index)"
      ]
    }
  }
}
```

## Enforcement levels

```json
{
  "critical-skill": {
    "enforcement": "block",
    "priority": "high"
  },
  "recommended-skill": {
    "enforcement": "suggest",
    "priority": "medium"
  },
  "optional-skill": {
    "enforcement": "optional",
    "priority": "low"
  }
}
```

## Tips

- Start broad keywords, narrow based on false positives
- High priority for critical skills only (limits context injection)
- Test patterns with `DEBUG=true` before deploying
- Version control skill-rules.json alongside hook code
