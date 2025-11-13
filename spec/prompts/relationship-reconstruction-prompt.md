# Relationship Reconstruction Prompt

## Обзор

**Тип**: Error Correction Prompt
**Критичность**: ⭐⭐ Низкая (вспомогательный)
**Стадия**: Graph Construction (Stage 3, on-demand)
**Файл**: `NodeRAG/utils/prompt/relationship_reconstraction.py`
**Температура**: 0.0 (детерминистическая обработка)
**Частота использования**: ~5% relationships

---

## Назначение

Relationship Reconstruction Prompt — это **вспомогательный промпт** для исправления некорректно форматированных relationships, извлечённых Text Decomposition Prompt. Когда relationship не содержит ровно 3 элемента (source, relation, target), этот промпт реконструирует правильный формат.

---

## Полный текст промпта (English)

```
You will be given a string containing tuples representing relationships between entities. The format of these relationships is incorrect and needs to be reconstructed. The correct format should be: 'ENTITY_A,RELATION_TYPE,ENTITY_B', where each tuple contains three elements: two entities and a relationship type. Your task is to reconstruct each relationship in the following format: {'source': 'ENTITY_A', 'relation': 'RELATION_TYPE', 'target': 'ENTITY_B'}. Please ensure the output follows this structure, accurately mapping the entities and relationships provided.
Incorrect relationships tuple string:{relationship}
```

---

## Полный текст промпта (Chinese)

```
你将获得一个包含实体之间关系的元组字符串。这些关系的格式是错误的，需要被重新构建。正确的格式应为：'实体A,关系类型,实体B'，每个元组应包含三个元素：两个实体和一个关系类型。你的任务是将每个关系重新构建为以下格式：{'source': '实体A', 'relation': '关系类型', 'target': '实体B'}。请确保输出遵循此结构，准确映射提供的实体和关系。
错误的关系元组:{relationship}
```

---

## Структура промпта

### Компоненты

| Компонент | Назначение |
|-----------|-----------|
| **Problem Statement** | Объяснение проблемы (некорректный формат) |
| **Correct Format** | Определение правильного формата |
| **Task Description** | Реконструкция в формат `{'source', 'relation', 'target'}` |
| **Input Placeholder** | `{relationship}` — некорректная строка |

---

## Structured Output Schema

```python
from pydantic import BaseModel

class relationship_reconstraction(BaseModel):
    source: str
    relationship: str
    target: str
```

**Файл**: `NodeRAG/utils/prompt/json_format.py:11-14`

---

## Примеры использования

### Случай 1: Только 2 элемента

**Вход (некорректный)**:
```
"DR. EMILY ROBERTS, attended conference in Paris"
```

**Проблема**: Только 2 элемента (entity и relation+entity).

**Выход (исправленный)**:
```json
{
  "source": "DR. EMILY ROBERTS",
  "relationship": "attended conference in",
  "target": "PARIS"
}
```

**Семантическая операция**: Разбиение второго элемента на relation и target.

### Случай 2: Больше 3 элементов

**Вход (некорректный)**:
```
"DR. EMILY ROBERTS, presented, research, on, SOLAR PANELS"
```

**Проблема**: 5 элементов (слишком много запятых).

**Выход (исправленный)**:
```json
{
  "source": "DR. EMILY ROBERTS",
  "relationship": "presented research on",
  "target": "SOLAR PANELS"
}
```

**Семантическая операция**: Объединение средних элементов в relation.

### Случай 3: Пропущенная entity

**Вход (некорректный)**:
```
"attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"
```

**Проблема**: Отсутствует source entity.

**Выход (исправленный)**:
```json
{
  "source": "UNKNOWN",
  "relationship": "attended",
  "target": "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"
}
```

**Примечание**: LLM может попытаться инференцировать source из контекста, если он был предоставлен.

---

## Роль в цепочке преобразований

### Позиция в Pipeline

```
Text Decomposition
      ↓
{semantic_units, entities, relationships}
      ↓
Format Validation
      ↓
  ❌ Invalid format detected
      ↓
[RELATIONSHIP RECONSTRUCTION PROMPT] ← ВЫ ЗДЕСЬ
      ↓
Fixed relationship
      ↓
Graph Construction
```

### Триггер

**Условие активации**:
```python
for relationship in relationships:
    parts = relationship.split(',')
    if len(parts) != 3:
        # Требуется реконструкция
        fixed = reconstruct_relationship(relationship)
```

**Частота**: ~5% всех relationships.

---

## Связи с другими промптами

### Upstream Prompt

#### Text Decomposition Prompt
**Связь**: Создаёт relationships, которые могут быть некорректными.

**Причины ошибок**:
1. LLM неправильно разместил запятые
2. Сложная структура предложения
3. Ambiguous relationship description

**Пример проблемы**:
```
Text: "Dr. Roberts, working at the Institute, presented research."

Text Decomposition может выдать:
  "DR. ROBERTS, working at the Institute, presented research"

Проблема: 3+ элемента, но неправильная структура
```

---

## Внешние библиотеки и инструменты

### LLM Providers

**Те же, что и Text Decomposition**:
- OpenAI GPT-4o-mini (рекомендуется)
- Google Gemini-1.5-Flash

**Параметры**:
```python
{
    "model": "gpt-4o-mini",
    "temperature": 0.0,
    "response_format": relationship_reconstraction
}
```

### Pydantic

**Schema**: `relationship_reconstraction`

```python
from pydantic import BaseModel

class relationship_reconstraction(BaseModel):
    source: str
    relationship: str
    target: str
```

---

## Метрики

### Стоимость

**Per reconstruction**:
```
Input: ~50 tokens (короткая строка)
Output: ~30 tokens (JSON)

Cost: ~$0.00001 (GPT-4o-mini)
```

**Для документа**:
```
Relationships: 100
Malformed: 5 (5%)
Total reconstruction cost: $0.00005 (пренебрежимо)
```

### Latency

- **Per relationship**: ~0.5-1 секунда
- **Impact на pipeline**: Минимальный (редко используется)

### Точность

- **Success rate**: >95% (LLM хорошо справляется с разбором)
- **Manual review**: Рекомендуется для критических данных

---

## Ограничения

### 1. Потеря семантики

**Проблема**: При реконструкции может быть потеря нюансов.

```
Оригинал: "DR. ROBERTS, presented groundbreaking research on, SOLAR PANELS"

Reconstruction может упростить:
  relation: "presented research on"

Потеря: "groundbreaking" (квалификатор)
```

### 2. Ambiguity в разбиении

**Проблема**: Неясно, где граница между relation и target.

```
Input: "DR. ROBERTS, conducted research in renewable energy for, INSTITUTE"

Возможные интерпретации:
1. relation="conducted research in renewable energy for", target="INSTITUTE"
2. relation="conducted research in", target="RENEWABLE ENERGY for INSTITUTE"

LLM выберет одну, не обязательно правильную.
```

---

## Лучшие практики

### 1. Предотвращение ошибок в источнике

Улучшить Text Decomposition Prompt:
```
"Please make sure EACH relationship string contains EXACTLY three elements
separated by commas: source entity, relation type, target entity."
```

### 2. Валидация перед реконструкцией

```python
# Проверить, можно ли исправить без LLM
parts = relationship.split(',')
if len(parts) == 2:
    # Простой случай: попробовать разбить второй элемент
    # ...
elif len(parts) > 3:
    # Сложный случай: использовать LLM
    fixed = reconstruct_relationship(relationship)
```

### 3. Логирование

```python
logger.warning(
    f"Malformed relationship: {relationship}\n"
    f"Reconstructed: {fixed}"
)
```

---

## Заключение

Relationship Reconstruction Prompt — **вспомогательный инструмент** для поддержания качества данных. Хотя его использование редкое (~5%), он критичен для обеспечения целостности графа знаний.

**Ключевые характеристики**:
- ✅ Автоматическое исправление ошибок формата
- ✅ Минимальная стоимость и latency
- ✅ Высокая точность (>95%)
- ⚠️ Может упростить сложные relationships

---

## Ссылки

- **Файл промпта**: `NodeRAG/utils/prompt/relationship_reconstraction.py:1-10`
- **Schema**: `NodeRAG/utils/prompt/json_format.py:11-14`
- **Manager**: `NodeRAG/utils/prompt/prompt_manager.py:29-36, 88-89`
- **Upstream**: Text Decomposition Prompt
- **Downstream**: Graph Construction
