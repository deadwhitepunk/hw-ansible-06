# Домашнее задание к занятию 6 «Создание собственных модулей»

## Подготовка к выполнению

1. Создайте пустой публичный репозиторий в своём любом проекте: `my_own_collection`.
2. Скачайте репозиторий Ansible: `git clone https://github.com/ansible/ansible.git` по любому, удобному вам пути.
3. Зайдите в директорию Ansible: `cd ansible`.
4. Создайте виртуальное окружение: `python3 -m venv venv`.
5. Активируйте виртуальное окружение: `. venv/bin/activate`. Дальнейшие действия производятся только в виртуальном окружении.
6. Установите зависимости `pip install -r requirements.txt`.
7. Запустите настройку окружения `. hacking/env-setup`.
8. Если все шаги прошли успешно — выйдите из виртуального окружения `deactivate`.
9. Ваше окружение настроено. Чтобы запустить его, нужно находиться в директории `ansible` и выполнить конструкцию `. venv/bin/activate && . hacking/env-setup`.

## Основная часть

Ваша цель — написать собственный module, который вы можете использовать в своей role через playbook. Всё это должно быть собрано в виде collection и отправлено в ваш репозиторий.

**Шаг 1.** В виртуальном окружении создайте новый `my_own_module.py` файл.

**Шаг 2.** Наполните его содержимым:

**Шаг 3.** Заполните файл в соответствии с требованиями Ansible так, чтобы он выполнял основную задачу: module должен создавать текстовый файл на удалённом хосте по пути, определённом в параметре `path`, с содержимым, определённым в параметре `content`.

```python
import hashlib
import os
from pathlib import Path
from ansible.module_utils.basic import AnsibleModule

def calculate_sha256(file_path):
    with open(file_path, "rb") as file:
        content = file.read()
    return hashlib.sha256(content).hexdigest()

def create_file(path, name, content):
    path_to_file = Path(path) / name
    result = {"changed": False, "message": ""}
    Path(path).mkdir(parents=True, exist_ok=True)
    if path_to_file.exists():
        old_digest = calculate_sha256(path_to_file)    
        tmp_dir = Path.home() / 'tmp'
        tmp_dir.mkdir(parents=True, exist_ok=True)
        tmp_path = tmp_dir / name
        
        with open(tmp_path, 'w') as f:
            f.write(content)
        
        new_digest = calculate_sha256(tmp_path)
        os.remove(tmp_path)
        
        if old_digest != new_digest:
            with open(path_to_file, 'w') as f:
                f.write(content)
            result["changed"] = True
            result["message"] = "File updated"
        else:
            result["message"] = "File unchanged"
    else:
        with open(path_to_file, 'w') as f:
            f.write(content)
        result["changed"] = True
        result["message"] = "File created"
    
    return result

def remove_file(path, name):
    path_to_file = Path(path) / name
    result = {"changed": False, "message": ""}
    
    if path_to_file.exists():
        os.remove(path_to_file)
        result["changed"] = True
        result["message"] = "File removed"
    else:
        result["message"] = "File does not exist"
    
    return result

def main():
    module_args = dict(
        name=dict(type='str', required=True),
        path=dict(type='str', required=True),
        content=dict(type='str', required=True),
        state=dict(type='str', required=True, choices=['present', 'absent'])
    )

    result = dict(
        changed=False,
        original_message='',
        message=''
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    name = module.params['name']
    path = module.params['path']
    content = module.params['content']
    state = module.params['state']

    result['original_message'] = {
        'name': name,
        'path': path,
        'state': state
    }

    # Check mode
    if module.check_mode:
        result['message'] = f"Would {'create' if state == 'present' else 'remove'} file {name} at {path}"
        module.exit_json(**result)

    try:
        if state == 'present':
            file_result = create_file(path, name, content)
            result['changed'] = file_result['changed']
            result['message'] = file_result['message']
        elif state == 'absent':
            file_result = remove_file(path, name)
            result['changed'] = file_result['changed']
            result['message'] = file_result['message']
        
        module.exit_json(**result)
    
    except Exception as e:
        module.fail_json(msg=f"Error: {str(e)}", **result)

if __name__ == '__main__':
    main()
```

**Шаг 4.** Проверьте module на исполняемость локально.

![Manual start](https://github.com/deadwhitepunk/hw-ansible-06/blob/main/img/ansible_manual_start.png)

**Шаг 5.** Напишите single task playbook и используйте module в нём.

![Single task](https://github.com/deadwhitepunk/hw-ansible-06/blob/main/img/ansible_single_playbook.png)

**Шаг 6.** Проверьте через playbook на идемпотентность.

Проверка на идемпотентность пройдена успешно, так как ansible отправил нам "ок", это значит что никаких изменений внесено не было и файл остался неизменным.

![iden](https://github.com/deadwhitepunk/hw-ansible-06/blob/main/img/ansible_idenp_ok.png)

**Шаг 7.** Выйдите из виртуального окружения.

`"deactivate"`

**Шаг 8.** Инициализируйте новую collection: `ansible-galaxy collection init my_own_namespace.yandex_cloud_elk`.

**Шаг 9.** В эту collection перенесите свой module в соответствующую директорию.

![Collection with module](https://github.com/deadwhitepunk/hw-ansible-06/blob/main/img/galaxy_with_my_module.png)

**Шаг 10.** Single task playbook преобразуйте в single task role и перенесите в collection. У role должны быть default всех параметров module.
```sh
 ansible-galaxy role init role_create_file
- Role role_create_file was created successfully
```

tasks/main.yaml
```yaml
---
- name: Create file
  my_file_creator:
      name: "{{ name_of_file }}"
      path: "{{ path_to_file }}"
      content: "{{ content }}"
      state: "{{ state }}"
```

vars/main.yaml
```yaml
---
# vars file for role_create_file
name_of_file: "test.txt"
path_to_file: "/home/sadmin"
content: "Hello world!"
state: "present"
```

**Шаг 11.** Создайте playbook для использования этой role.

```yaml
---
- name: Test module
  hosts: localhost
  roles:
    - role: role_create_file
```
**Шаг 12.** Заполните всю документацию по collection, выложите в свой репозиторий, поставьте тег `1.0.0` на этот коммит.

[Ссылка на репозиторий с коллекцией](https://github.com/deadwhitepunk/yandex_cloud_elk)

[Ссылка на тэг](https://github.com/deadwhitepunk/yandex_cloud_elk/releases/tag/1.0.0)

**Шаг 13.** Создайте .tar.gz этой collection: `ansible-galaxy collection build` в корневой директории collection.
```sh
❯ ansible-galaxy collection build
Created collection for my_own_namespace.yandex_cloud_elk at /home/sadmin/netology-hw/ANSIBLE/my_own_namespace/yandex_cloud_elk/my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
```
**Шаг 14.** Создайте ещё одну директорию любого наименования, перенесите туда single task playbook и архив c collection.

![New directory](https://github.com/deadwhitepunk/hw-ansible-06/blob/main/img/new_dir.png)

**Шаг 15.** Установите collection из локального архива: `ansible-galaxy collection install <archivename>.tar.gz`.

![Install collection](https://github.com/deadwhitepunk/hw-ansible-06/blob/main/img/install_collection.png)

**Шаг 16.** Запустите playbook, убедитесь, что он работает.

![Install collection](https://github.com/deadwhitepunk/hw-ansible-06/blob/main/img/success_playbook_from_collection.png)

**Шаг 17.** В ответ необходимо прислать ссылки на collection и tar.gz архив, а также скриншоты выполнения пунктов 4, 6, 15 и 16.

[Ссылка на репозиторий с коллекцией](https://github.com/deadwhitepunk/yandex_cloud_elk)

[TAR GZ ARCHIVE](https://github.com/deadwhitepunk/yandex_cloud_elk/blob/main/new_dir/my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz)

---