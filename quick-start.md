# Руководство по быстрому старту GitPython

Добро пожаловать в руководство по быстрому старту GitPython! Этот краткий ресурс предназначен для разработчиков, которые ищут практический и интерактивный опыт обучения, и предлагает пошаговые фрагменты кода для быстрой инициализации/клонирования репозиториев, выполнения основных операций Git и изучения возможностей GitPython. Приготовьтесь погрузиться в работу, экспериментировать и раскрыть силу GitPython в ваших проектах!

## git.Repo

Существует несколько способов создания объекта `git.Repo`

### Инициализировать новый git-репозиторий

```python
# $ git init <path/to/dir>

from git import Repo

repo = Repo.init(path_to_dir)
```

### Использовать существующий локальный git-репозиторий

```python
repo = Repo(path_to_dir)
```

### Клонировать из URL

На протяжении оставшейся части этого руководства мы будем использовать клон репозитория <https://github.com/gitpython-developers/QuickStartTutorialFiles.git>

```python
# $ git clone <url> <local_dir>

repo_url = "https://github.com/gitpython-developers/QuickStartTutorialFiles.git"

repo = Repo.clone_from(repo_url, local_dir)
```

## Деревья и Blob-объекты

### Дерево последнего коммита (Latest Commit Tree)

Это структура каталогов и файлов репозитория в состоянии, соответствующем последнему коммиту.

```python
tree = repo.head.commit.tree
```

### Дерево любого коммита (Any Commit Tree)

Это структура каталогов и файлов репозитория в состоянии, соответствующем конкретному коммиту (не обязательно последнему).

```python
prev_commits = list(repo.iter_commits(all=True, max_count=10))  # последние 10 коммитов из всех веток.
tree = prev_commits[0].tree
```

### Отображение содержимого 1-ого уровня (корневой директории репозитория)

```python
files_and_dirs = [(entry, entry.name, entry.type) for entry in tree]
files_and_dirs

# Output
# [(< git.Tree "SHA1-HEX_HASH" >, 'Downloads', 'tree'),
#  (< git.Tree "SHA1-HEX_HASH" >, 'dir1', 'tree'),
#  (< git.Blob "SHA1-HEX_HASH" >, 'file4.txt', 'blob')]
```

### Рекурсивный обход дерева

```python
def print_files_from_git(root, level=0):
    for entry in root:
        print(f"{'-' * 4 * level}| {entry.path}, {entry.type}")
        if entry.type == "tree":
            print_files_from_git(entry, level + 1)
```

```python
print_files_from_git(tree)

# Output
# | Downloads, tree
# ----| Downloads / file3.txt, blob
# | dir1, tree
# ----| dir1 / file1.txt, blob
# ----| dir1 / file2.txt, blob
# | file4.txt, blob
```

## Использование

### Добавление файлов в Индекс

```python
# Для начала необходимо внести изменения в файл, чтобы иметь возможность добавить обновление в Git

update_file = "dir1/file2.txt"  # используем файл local_dir/dir1/file2.txt
with open(f"{local_dir}/{update_file}", "a") as f:
    f.write("\nUpdate version 2")
```

Теперь давайте добавим обновленный файл в Git

```python
# $ git add <file>
add_file = [update_file]  # относительный путь от корня git
repo.index.add(add_file)  # передаем список путей в функцию add
```
 > Обратите внимание, что функция add требует список путей, даже если вы добавляете всего один файл.

**Предупреждение:** Если у вас возникли проблемы с этим, попробуйте вместо этого вызвать `git` напрямую через `repo.git.add(path)`

- `repo.index.add()` — использует внутреннюю реализацию GitPython
- `repo.git.add()` — выполняет настоящую git-команду через командную строку, что гарантирует такое же поведение, как при ручном запуске `git add`

### Коммит (Commit)

```python
# $ git commit -m <message>
repo.index.commit("Update to file2")
```

### Список коммитов, связанных с файлом

```python
# $ git log <file>

# Относительный путь от корня git-репозитория
repo.iter_commits(all=True, max_count=10, paths=update_file)  # Получает последние 10 коммитов из всех веток.

# Output: <generator object Commit._iter_from_process_or_stream at 0x7fb66c186cf0>
```

Обратите внимание, что возвращается объект-генератор

```python
commits_for_file_generator = repo.iter_commits(all=True, max_count=10, paths=update_file)
commits_for_file = list(commits_for_file_generator)
commits_for_file

# Output:
# [<git.Commit "SHA1-HEX_HASH-2">,
#  <git.Commit "SHA1-HEX-HASH-1">]
```

возвращает список объектов `Commit`

### Вывод текстовых файлов

Давайте выведем последнюю версию файла `<local_dir>/dir1/file2.txt`

```python
print_file = "dir1/file2.txt"
tree[print_file]  # Дерево коммита, на который указывает HEAD

# Output <git.Blob "SHA1-HEX-HASH">
```

```python
blob = tree[print_file]
print(blob.data_stream.read().decode())

# Output
# File 2 version 1
# Update version 2
```

Предыдущая версия файла `<local_dir>/dir1/file2.txt`

```python
commits_for_file = list(repo.iter_commits(all=True, paths=print_file))
tree = commits_for_file[-1].tree  # Получает дерево первого коммита.
blob = tree[print_file]

print(blob.data_stream.read().decode())

# Output
# File 2 version 1
```

### Статус (Status)

- Неотслеживаемые файлы

    Давайте создадим новый файл

    ```python
    f = open(f"{local_dir}/untracked.txt", "w")  # Создает пустой файл
    f.close()
    ```

    ```python
    repo.untracked_files
    # Output: ['untracked.txt']
    ```

- Измененные файлы

    ```python
    # Давайте изменим один из отслеживаемых файлов.

    with open(f"{local_dir}/Downloads/file3.txt", "w") as f:
        f.write("file3 version 2")  # Перезаписали file 3.
    ```

    ```python
    repo.index.diff(None)  # Сравнивает файлы в Индексе с рабочей директорией

    # Output:
    № [<git.diff.Diff object at 0x7fb66c076e50>,
    #  <git.diff.Diff object at 0x7fb66c076ca0>]
    ```

    возвращает список сравниваемых (`diff`) объектов
    
    ```python
    diffs = repo.index.diff(None)
    for d in diffs:
        print(d.a_path)

    # Output
    # Downloads/file3.txt
    ```

### Сравнения (Diffs)

Сравнивает Индекс с последним коммитом

```python
diffs = repo.index.diff(repo.head.commit)
for d in diffs:
    print(d.a_path)

# Output

```

```python
# Давайте добавим неотслеживаемый файл untracked.txt.
repo.index.add(["untracked.txt"])
diffs = repo.index.diff(repo.head.commit)
for d in diffs:
    print(d.a_path)

# Output
# untracked.txt
```

Сравнивает коммит с коммитом

```python
first_commit = list(repo.iter_commits(all=True))[-1]
diffs = repo.head.commit.diff(first_commit)
for d in diffs:
    print(d.a_path)

# Output
# dir1/file2.txt
```

## Дополнительные ресурсы

Помните, это только начало! С GitPython вы можете достичь гораздо большего в вашем рабочем процессе разработки. Чтобы исследовать дальнейшие возможности и открыть для себя расширенные функции, ознакомьтесь с [полным руководством GitPython](tutorial.md) и [справочником API](api-reference.md). Удачного программирования!
