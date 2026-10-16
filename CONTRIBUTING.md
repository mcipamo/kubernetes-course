# 🤝 Contributing

Thanks for helping improve the course! Every contribution is welcome: fixing a typo, clarifying an
analogy, updating a command or proposing a new module.

## Content rules

| Rule | Detail |
|---|---|
| Source | Every technical fact must be verifiable at https://kubernetes.io/docs/ |
| Version | The course targets Kubernetes **v1.37**; always label alpha/beta features |
| Structure | Every module has `README.md`, `LAB.md` and `EXERCISES.md` |
| Style | Plain English, second person, short paragraphs, analogy before jargon |
| Labs | Must work on the `kc` kind cluster from module 03 |
| Solutions | Always inside `<details>` so they do not spoil the exercise |

## Workflow

1. Open an issue describing the change.
2. Fork, create a branch (`fix/module-08-dns`) and make the change.
3. Test the affected lab on your kind cluster.
4. Open a pull request linking the issue and the official page that backs up the change.
