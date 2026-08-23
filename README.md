# GUIA COMPLETO DE ESTUDO — CELERY + DJANGO

Guia definitivo explicando estrutura, configuração, funcionamento interno e arquitetura profissional usando Celery + Django com Redis.

---

# ESTRUTURA DO PROJETO

```
meu_projeto/
│
├── meu_projeto/
│   ├── __init__.py
│   ├── settings.py
│   ├── celery.py
│
├── usuarios/
│   ├── views.py
│   ├── tasks.py
│
└── manage.py
```

---

# ARQUIVO: meu_projeto/celery.py

```python
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "meu_projeto.settings")

app = Celery("meu_projeto")

app.config_from_object("django.conf:settings", namespace="CELERY")

app.autodiscover_tasks()
```

## Explicação detalhada

- os → permite acessar variáveis de ambiente.
- Celery → classe principal que cria a aplicação Celery.
- DJANGO_SETTINGS_MODULE → informa ao Celery onde está o settings.py.
- Celery("meu_projeto") → nome da aplicação (usado no comando -A).
- config_from_object() → carrega todas as configurações que começam com CELERY_.
- autodiscover_tasks() → procura automaticamente arquivos tasks.py nas apps.

---

# ARQUIVO: meu_projeto/__init__.py

```python
from .celery import app as celery_app

__all__ = ("celery_app",)
```

## Explicação

- Garante que o Celery seja carregado quando o Django iniciar.
- Sem isso, as tarefas podem não ser registradas.
- Conecta oficialmente Django ↔ Celery.

---

# ARQUIVO: settings.py

```python
CELERY_BROKER_URL = "redis://localhost:6379/0"
CELERY_RESULT_BACKEND = "redis://localhost:6379/0"

CELERY_ACCEPT_CONTENT = ["json"]
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"

CELERY_TIMEZONE = "UTC"
```

## Explicação

- Broker → intermediário (Redis).
- Redis porta 6379 banco 0.
- RESULT_BACKEND → onde o resultado da tarefa é salvo.
- ACCEPT_CONTENT → aceita apenas JSON (segurança).
- SERIALIZER → converte Python ↔ JSON.
- TIMEZONE → fuso horário para tarefas agendadas.

---

# ARQUIVO: usuarios/tasks.py

```python
from celery import shared_task
import requests

@shared_task(bind=True, max_retries=3)
def enviar_email_usuario(self, usuario_id):
    try:
        print(f"Processando usuário {usuario_id}")

        resposta = requests.get("https://httpbin.org/get")

        return {
            "status": "sucesso",
            "usuario": usuario_id
        }

    except Exception as e:
        raise self.retry(exc=e, countdown=5)
```

## Explicação

- shared_task → registra a função como tarefa do Celery.
- bind=True → permite acessar self.
- max_retries=3 → tenta novamente até 3 vezes.
- self.retry() → se falhar, espera 5 segundos e tenta novamente.
- Ideal para APIs externas, pagamentos, envio de email e processamentos demorados.

---

# ARQUIVO: usuarios/views.py

```python
from django.http import JsonResponse
from .tasks import enviar_email_usuario

def enviar_email(request):

    enviar_email_usuario.apply_async(
        args=[1],
        countdown=30
    )

    return JsonResponse({"mensagem": "Tarefa agendada"})
```

## Explicação

- apply_async() → versão avançada do delay().
- args=[1] → 1 vira o parametro usuario_id.
- countdown=30 → executa após 30 segundos.
- A view responde imediatamente enquanto o processamento roda em segundo plano.

---

# FLUXO COMPLETO DO SISTEMA

1) View chama apply_async()
2) Celery serializa a tarefa
3) Envia para o Redis (Broker)
4) Worker pega da fila
5) Executa
6) Se falhar → retry automático
7) Salva resultado

---

# PAPEL DE CADA ARQUIVO

celery.py   → Inicializa Celery
__init__.py → Conecta Celery ao Django
settings.py → Configura Redis
tasks.py    → Define tarefas e retry
views.py    → Dispara tarefas

---

# ARQUITETURA PROFISSIONAL

Producer → Django
Broker   → Redis
Consumer → Worker

Esse padrão é chamado de sistema distribuído baseado em filas.

---

# COMANDO PARA RODAR WORKER

```bash
celery -A meu_projeto worker --loglevel=info
```

---

# CONCEITOS DOMINADOS

- Integração Celery + Django
- Broker
- Worker
- Retry automático
- Serialização
- `apply_async()`
- Arquitetura distribuída

---

# CONCLUSÃO

Você configurou um sistema assíncrono profissional capaz de:

- Executar tarefas em segundo plano
- Reprocessar falhas automaticamente
- Escalar horizontalmente
- Desacoplar processamento pesado da aplicação web

Esse é o padrão usado em:

- Processamento de pagamentos
- Envio de e-mails em massa
- Processamento de imagens
- Integrações com APIs externas
- Filas de microserviços

---

## Autor

**Orlando Conceição Vilhalba de Almeida**

Desenvolvedor Backend em formação, com foco em Python, Django, Django REST Framework, PostgreSQL, APIs REST e Docker, utilizando React como tecnologia complementar para integração das aplicações.

GitHub: https://github.com/orlandoconceicao

LinkedIn: https://www.linkedin.com/in/orlando-concei%C3%A7%C3%A3o-582234315

Portfólio: https://orlandoconceicao.github.io/
