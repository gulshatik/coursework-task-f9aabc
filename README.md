# Сырой текст задания → плоская карточка

## Задание

Разработать модуль, который принимает **неформальное описание задачи** (как в переписке или на сайте курса) и преобразует его в **структурированный объект**, пригодный для дальнейшей обработки в коде.  
Вход – одна строка или короткий абзац.  
Выход – объект с полями: `title`, `subject`, `deadline_hint`, `deliverable_type`, `grading_hints`.

> **Важно**: модель должна вернуть данные **одним ответом** – без уточняющих вопросов и диалога.

---

## Контекст проекта

В рамках курса вы создаёте «ассистента по учёбе».  
Он должен принимать неформальное описание задания и выдавать **проверяемый набор полей**, который можно фильтровать, сохранять и передавать дальше в пайплайн.

---

## Цель

Научиться из **одного** пользовательского текста получить **проверяемый** набор полей без диалога и без ручного разбора строки в Python.

---

## Стек и установка

| Пакет | Версия | Команда установки |
|-------|--------|-------------------|
| `langchain-core` | 0.2+ | `pip install langchain-core` |
| `langchain-openai` | 0.2+ | `pip install langchain-openai` |
| `pydantic` | 2.0+ | `pip install pydantic` |
| `python-dotenv` | 1.0+ | `pip install python-dotenv` |

```bash
pip install langchain-core langchain-openai pydantic python-dotenv
```

> **Подключение к модели**: через переменную окружения `OPENAI_API_KEY` (или через настройки клиента). Конкретную модель в задании не фиксируем.

---

## Условия сценария

| Параметр | Требование |
|----------|------------|
| Вход | **Одна** строка или короткий абзац с формулировкой задания от «преподавателя» (выдуманная). |
| Выход | Объект с полями: `title`, `subject`, `deadline_hint`, `deliverable_type`, `grading_hints`. |
| Диалог | **Нет** уточняющих вопросов. Модель делает один проход. |
| Структура | Один уровень списка (`grading_hints` – список строк). Нет вложенных деревьев. |

---

## Что нужно сделать

1. **Определить Pydantic‑модель** `TaskCard` с полями:
   - `title: str` – название задания.
   - `subject: str` – предмет/тема.
   - `deadline_hint: str` – свободная строка с указанием срока.
   - `deliverable_type: str` – тип сдачи (отчёт, код, презентация и т.д.).
   - `grading_hints: List[str]` – список ключевых слов/фраз, упомянутых в описании оценки.

2. **Собрать цепочку**:
   - Шаблон промпта → вызов LLM → парсер, который гарантирует тип результата `BaseModel`.
   - В промпте явно попросить модель вернуть данные в формате, который ожидает ваш парсер.

3. **Вывести в консоль**:
   - Валидированный объект (`model_dump()`).
   - Краткую человекочитаемую сводку.

---

## Подсказки

```python
from pydantic import BaseModel, Field
from typing import List

class TaskCard(BaseModel):
    title: str = Field(..., description="Название задания")
    subject: str = Field(..., description="Тема/предмет")
    deadline_hint: str = Field(..., description="Указание срока сдачи")
    deliverable_type: str = Field(..., description="Тип сдачи (отчёт, код, презентация)")
    grading_hints: List[str] = Field(..., description="Ключевые слова оценки")
```

```python
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import PydanticOutputParser

parser = PydanticOutputParser(pydantic_object=TaskCard)

prompt_template = """
You are an assistant that extracts structured data from a task description.
The input is a single string containing a task description in Russian.
Return the data in the following JSON format:

{format_instructions}

Input:
{raw_text}
"""

prompt = PromptTemplate(
    template=prompt_template,
    input_variables=["raw_text"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)

llm = ChatOpenAI(temperature=0)

chain = prompt | llm | parser
```

---

## Ожидаемый результат

**Пример входа**

```
Сдайте к пятнице мини-отчёт по LangChain: 2 страницы, упор на агентов. Оценка: за полноту и за пример кода.
```

**Пример вывода**

```text
Parsed object:
{'title': 'мини-отчёт', 'subject': 'LangChain', 'deadline_hint': 'к пятнице', 'deliverable_type': 'отчёт', 'grading_hints': ['полнота', 'пример кода']}

Summary:
Task: мини-отчёт
Subject: LangChain
Deadline: к пятнице
Deliverable: отчёт
Grading hints: полнота, пример кода
```

---

## Критерии оценки

| Критерий | Баллы |
|----------|-------|
| Модель данных отражает сценарий (поля + типы) | 2 |
| Один запрос — один проход по цепочке, без лишнего диалога | 2 |
| Используется парсер структуры, а не «разбор руками» | 2 |
| Промпт согласован с инструкциями формата | 2 |
| Вывод: и структура, и краткая сводка | 2 |
| **Итого** | **10** |

---

## Документация

- [Structured output (LangChain)](https://docs.langchain.com/oss/python/langchain/structured-output)

---

## Ожидания преподавателя

| Пункт | Требование |
|-------|------------|
| **TASK_TYPE** | code |
| **STACK** | Python 3.10+, `langchain-core`, `langchain-openai`, `pydantic`, `python-dotenv` |
| **FILES** | `models.py`, `chain.py`, `main.py` |
| **RUN_COMMAND** | `python main.py` |
| **REQUIREMENTS_CHECKLIST** | Полный список, приведённый в разделе «Что нужно сделать» и «Подсказки». |
| **PLAN** | 1. `models.py` – Pydantic‑модель. <br>2. `chain.py` – Prompt, LLM, парсер, цепочка. <br>3. `main.py` – загрузка ключа, запуск цепочки, вывод. |

---

## План реализации

1. **Создать `models.py`**

   ```python
   # models.py
   from pydantic import BaseModel, Field
   from typing import List

   class TaskCard(BaseModel):
       title: str = Field(..., description="Название задания")
       subject: str = Field(..., description="Тема/предмет")
       deadline_hint: str = Field(..., description="Указание срока сдачи")
       deliverable_type: str = Field(..., description="Тип сдачи (отчёт, код, презентация)")
       grading_hints: List[str] = Field(..., description="Ключевые слова оценки")
   ```

2. **Создать `chain.py`**

   ```python
   # chain.py
   from langchain_core.prompts import PromptTemplate
   from langchain_openai import ChatOpenAI
   from langchain_core.output_parsers import PydanticOutputParser
   from models import TaskCard

   parser = PydanticOutputParser(pydantic_object=TaskCard)

   prompt_template = """
   You are an assistant that extracts structured data from a task description.
   The input is a single string containing a task description in Russian.
   Return the data in the following JSON format:

   {format_instructions}

   Input:
   {raw_text}
   """

   prompt = PromptTemplate(
       template=prompt_template,
       input_variables=["raw_text"],
       partial_variables={"format_instructions": parser.get_format_instructions()},
   )

   llm = ChatOpenAI(temperature=0)

   chain = prompt | llm | parser
   ```

3. **Создать `main.py`**

   ```python
   # main.py
   import os
   from dotenv import load_dotenv
   from chain import chain

   load_dotenv()  # загружает OPENAI_API_KEY

   if __name__ == "__main__":
       raw_text = (
           "Сдайте к пятнице мини-отчёт по LangChain: 2 страницы, упор на агентов. "
           "Оценка: за полноту и за пример кода."
       )

       try:
           result = chain.invoke({"raw_text": raw_text})
           print("Parsed object:")
           print(result.model_dump())
           print("\nSummary:")
           print(
               f"Task: {result.title}\n"
               f"Subject: {result.subject}\n"
               f"Deadline: {result.deadline_hint}\n"
               f"Deliverable: {result.deliverable_type}\n"
               f"Grading hints: {', '.join(result.grading_hints)}"
           )
       except Exception as e:
           print(f"Error parsing task: {e}")
   ```

4. **Опциональный `requirements.txt`**

   ```
   langchain-core
   langchain-openai
   pydantic
   python-dotenv
   ```

5. **Запуск**

   ```bash
   export OPENAI_API_KEY=sk-...
   python main.py
   ```

   Ожидаемый вывод (значения могут слегка отличаться в зависимости от модели):

   ```
   Parsed object:
   {'title': 'мини-отчёт', 'subject': 'LangChain', 'deadline_hint': 'к пятнице', 'deliverable_type': 'отчёт', 'grading_hints': ['полнота', 'пример кода']}

   Summary:
   Task: мини-отчёт
   Subject: LangChain
   Deadline: к пятнице
   Deliverable: отчёт
   Grading hints: полнота, пример кода
   ```

---

## Итоги

- **Модель данных** отражает все необходимые поля.
- **Один запрос** – один проход по цепочке, без диалога.
- **Парсер** гарантирует тип `TaskCard`.
- **Промпт** содержит явные инструкции по формату.
- **Вывод** включает валидированный объект и человекочитаемую сводку.

Удачной разработки!
