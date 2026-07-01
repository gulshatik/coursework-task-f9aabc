# Сырой текст задания → плоская карточка

## Задание

В рамках курса вы создаёте **ассистента по учёбе**: программа должна принимать неформальное описание задачи (как в переписке или на сайте курса) и превращать его в данные, с которыми дальше можно работать в коде (фильтровать, сохранять, передавать следующему шагу пайплайна).  
В этом упражнении вы реализуете **модуль разбора одной формулировки** в компактную сводку.

## Цель

Научиться из одного пользовательского текста получить проверяемый набор полей без диалога и без «ручного» разбора строки в Python.

## Стек и установка

- Python 3.10+
- Пакеты: `langchain-core`, `langchain-openai`, `pydantic` (версии — как рекомендует преподаватель).
- Подключение к модели через переменные окружения или настройки клиента (конкретную модель в задании не фиксируем).

```bash
pip install langchain-core langchain-openai pydantic python-dotenv
```

## Условия сценария

| Параметр | Требование |
|----------|------------|
| Вход | **Одна** строка или короткий абзац — формулировка задания от «преподавателя» (выдуманная). |
| Выход | Объект с полями: `title`, `subject`, `deadline_hint` (строка в свободной форме), `deliverable_type` (что сдать: отчёт, код, презентация…), `grading_hints` (список строк — что упомянуто про оценку). |
| Диалог | **Нет** уточняющих вопросов. Модель не ведёт переписку, только один ответ на один запрос. |
| Вложенность | Минимальная: один уровень списка, без деревьев и ссылок «узел → дети». |

## Что нужно сделать

1. Описать Pydantic‑модель карточки задания с полями выше (имена можно слегка изменить, смысл сохранить).
2. Собрать цепочку: шаблон промпта → вызов языковой модели → парсер, который гарантирует тип результата `BaseModel` (или эквивалент по соглашению курса).
3. В промпте явно попросить модель вернуть данные в формате, который ожидает ваш парсер.
4. Вывести в консоль и валидированный объект (`model_dump()`), а также краткую человекочитаемую сводку.

## Подсказки

```python
from pydantic import BaseModel, Field
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import PydanticOutputParser
```

### 1. Модель карточки задания

```python
class AssignmentCard(BaseModel):
    title: str = Field(..., description="Краткое название задания")
    subject: str = Field(..., description="Предмет/тема работы")
    deadline_hint: str = Field(
        ..., description="Указание срока сдачи в свободной форме"
    )
    deliverable_type: str = Field(
        ...,
        description="Тип сдаваемого материала (отчёт, код и т.д.)",
    )
    grading_hints: list[str] = Field(
        ...,
        description="Список критериев оценки, упомянутых преподавателем",
    )
```

### 2. Парсер

```python
parser = PydanticOutputParser(pydantic_object=AssignmentCard)
```

### 3. Шаблон промпта

```python
prompt = PromptTemplate(
    template="""
Преобразуйте следующее неформальное описание задания в JSON‑объект, соответствующий модели AssignmentCard.

{format_instructions}

Задание: "{assignment_text}"
""",
    input_variables=["assignment_text"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)
```

### 4. Модель LLM

```python
llm = ChatOpenAI(model="gpt-4o-mini")   # можно заменить на любую доступную модель
```

> **Важно**: ключ API берётся из переменной окружения `OPENAI_API_KEY`.  
> Для удобства создайте файл `.env` в корне проекта:
>
> ```dotenv
> OPENAI_API_KEY=sk-...
> ```

### 5. Сборка цепочки

```python
chain = prompt | llm | parser
```

### 6. Запуск и вывод результата

```python
if __name__ == "__main__":
    raw_text = (
        "Сдайте к пятнице мини‑отчёт по LangChain: 2 страницы, упор на агентов. "
        "Оценка: за полноту и за пример кода."
    )
    result = chain.invoke({"assignment_text": raw_text})

    print("Validated object:")
    print(result.model_dump())

    summary = (
        f"Задание \"{result.title}\" по теме {result.subject}. "
        f"Срок: {result.deadline_hint}. Тип сдачи: {result.deliverable_type}. "
        f"Критерии оценки: {', '.join(result.grading_hints)}."
    )
    print("\nКраткая сводка:")
    print(summary)
```

## Ожидаемый результат

При запуске `python main.py` вы должны увидеть что‑то вроде:

```text
Validated object:
{
  "title": "Мини-отчёт по LangChain",
  "subject": "LangChain",
  "deadline_hint": "к пятнице",
  "deliverable_type": "отчёт",
  "grading_hints": ["полнота", "пример кода"]
}

Краткая сводка:
Задание "Мини-отчёт по LangChain" по теме LangChain. Срок: к пятнице. Тип сдачи: отчёт. Критерии оценки: полнота, пример кода.
```

## Критерии оценки

| Критерий | Баллы |
|----------|-------|
| Модель данных отражает сценарий (поля + типы) | 2 |
| Один запрос — один проход по цепочке, без лишнего диалога | 2 |
| Используется парсер структуры, а не «разбор руками» | 2 |
| Промпт согласован с инструкциями формата | 2 |
| Вывод: и структура, и краткая сводка | 2 |
| **Итого** | **10** |

## Документация

- [Structured output (LangChain)](https://docs.langchain.com/oss/python/langchain/structured-output)

---

### Файлы проекта

```
project/
├── main.py          # точка входа, реализует цепочку
├── .env             # (необязательно) хранит OPENAI_API_KEY
└── requirements.txt # список зависимостей
```

#### `requirements.txt`

```text
langchain-core>=0.2.0
langchain-openai>=0.1.0
pydantic>=2.0
python-dotenv>=1.0
```

## Как запустить

```bash
# 1. Создайте виртуальное окружение (необязательно, но рекомендуется)
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# 2. Установите зависимости
pip install -r requirements.txt

# 3. Добавьте ваш ключ OpenAI в файл .env (или экспортируйте переменную)
echo "OPENAI_API_KEY=sk-..." > .env

# 4. Запустите скрипт
python main.py
```

> Если вы используете другую модель, поменяйте строку `model="gpt-4o-mini"` на нужную вам.

## Что дальше?

После успешного выполнения задания можно расширить функциональность:
- Добавить поддержку нескольких входных строк (batch‑обработка).
- Внедрить более сложные правила парсинга (например, распознавание дат в разных форматах).
- Интегрировать с системой управления задачами (Jira, Trello и т.д.).

Удачной разработки!
