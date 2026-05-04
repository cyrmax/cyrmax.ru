---
title: "Интерактивное обновление пакетов WinGet через PowerShell"
date: 2026-04-19
lastmod: 2026-05-04
description: "Небольшой PowerShell-скрипт для интерактивного выбора пакетов WinGet, которые нужно обновить или удалить."
tags: ["windows", "powershell", "администрирование"]
draft: false
---

## Суть

WinGet удобен для установки и обновления программ в Windows, но при регулярном использовании быстро начинает не хватать более гибкого интерактивного режима.

Например, хочется посмотреть список доступных обновлений и для каждого пакета выбрать действие:

- обновить;
- пропустить;
- пропустить все оставшиеся;

Для этого я написал небольшой PowerShell-скрипт.

## Требования

Скрипт рассчитан на PowerShell 7 и требует установленного модуля для работы с WinGet:

```powershell
Install-Module -Name 'Microsoft.WinGet.Client'
````

После установки модуля можно запускать сам скрипт.

## Что делает скрипт

Скрипт получает список пакетов, для которых доступны обновления, а затем по очереди спрашивает, что сделать с каждым пакетом.

Доступные действия:

| Действие   | Что делает                           |
| ---------- | ------------------------------------ |
| `Update`   | Добавляет пакет в очередь обновления |
| `Skip`     | Пропускает текущий пакет             |
| `Skip All` | Пропускает все оставшиеся пакеты     |
| `Remove`   | Удаляет текущий пакет                |

> Действие `Remove` я добавил для себя, потому что после обновления Windows параллельно чистил систему от ненужных программ. Если оно не нужно, эту ветку можно удалить из скрипта.

## Код

```powershell
$ErrorActionPreference = "Stop"
Set-StrictMode -Version Latest

Write-Host "Looking for available WinGet updates..."

$allUpdates = Get-WinGetPackage | Where-Object { $_.IsUpdateAvailable }

if (-not $allUpdates) {
    Write-Host "No available WinGet updates found"
    exit
}

Write-Host "$($allUpdates.Count) updatable packages found."

$packagesToUpgrade = @()
$skipRemaining = $false

foreach ($pkg in $allUpdates) {
    if ($skipRemaining) {
        break
    }

    Write-Host ""
    Write-Host "Package: $($pkg.Name)"
    Write-Host "Id: $($pkg.Id)"
    Write-Host "Installed version: $($pkg.InstalledVersion)"
    Write-Host "Available version: $($pkg.AvailableVersions[0])"

    $choices = @(
        New-Object System.Management.Automation.Host.ChoiceDescription "&Update", "Add to upgrade queue"
        New-Object System.Management.Automation.Host.ChoiceDescription "&Skip", "Skip this package"
        New-Object System.Management.Automation.Host.ChoiceDescription "Skip &All", "Skip all remaining packages"
        New-Object System.Management.Automation.Host.ChoiceDescription "&Remove", "Remove this package now"
    )

    $answer = $host.UI.PromptForChoice(
        "Action for $($pkg.Name)",
        "Choose an action",
        $choices,
        0
    )

    switch ($answer) {
        0 {
            $packagesToUpgrade += $pkg
        }

        1 {
            Write-Host "Skipping"
        }

        2 {
            Write-Host "Skipping all remaining packages"
            $skipRemaining = $true
        }

        3 {
            Write-Host "Removing $($pkg.Name)"

            try {
                Uninstall-WinGetPackage -Id $pkg.Id -ErrorAction Stop
            }
            catch {
                Write-Host "Removal for $($pkg.Id) failed"
                Write-Host "Reason: $($_.Exception.Message)"
            }
        }
    }
}

if ($packagesToUpgrade.Count -eq 0) {
    Write-Host "No packages were selected for upgrade"
    exit
}

Write-Host ""
Write-Host "Upgrading $($packagesToUpgrade.Count) packages..."

foreach ($pkg in $packagesToUpgrade) {
    Write-Host "Upgrading $($pkg.Name)"

    try {
        Update-WinGetPackage -Id $pkg.Id -ErrorAction Stop
    }
    catch {
        Write-Host "Upgrade for $($pkg.Id) failed"
        Write-Host "Reason: $($_.Exception.Message)"
    }
}
```

## Как использовать

1. Установить модуль для работы с WinGet:

    ```powershell
    Install-Module -Name 'Microsoft.WinGet.Client'
    ```

2. Сохранить скрипт, например в файл:

    ```text
    Update-WinGetPackages.ps1
    ```

3. Запустить его в PowerShell 7:

    ```powershell
    .\Update-WinGetPackages.ps1
    ```

4. Для каждого найденного обновления выбрать нужное действие.

## Что можно упростить

Если удаление пакетов не нужно, можно убрать этот пункт из массива `$choices`:

```powershell
New-Object System.Management.Automation.Host.ChoiceDescription "&Remove", "Remove this package now"
```

И удалить ветку:

```powershell
3 {
    Write-Host "Removing $($pkg.Name)"

    try {
        Uninstall-WinGetPackage -Id $pkg.Id -ErrorAction Stop
    }
    catch {
        Write-Host "Removal for $($pkg.Id) failed"
        Write-Host "Reason: $($_.Exception.Message)"
    }
}
```

После этого останется только выбор между обновлением и пропуском пакетов.

## Возможные проблемы

### Модуль не установлен

Если PowerShell не знает команды вроде `Get-WinGetPackage` или `Update-WinGetPackage`, сначала установите модуль:

```powershell
Install-Module -Name 'Microsoft.WinGet.Client'
```

### Скрипты запрещены политикой выполнения

Если PowerShell не разрешает запуск `.ps1`-файлов, можно временно запустить скрипт так:

```powershell
pwsh.exe -ExecutionPolicy Bypass -File .\Update-WinGetPackages.ps1
```

Или для старого Windows PowerShell (если заработает):

```powershell
powershell.exe -ExecutionPolicy Bypass -File .\Update-WinGetPackages.ps1
```

### Некоторые пакеты не обновляются

Это нормально: не все пакеты идеально взаимодействуют с WinGet. Тема подобных менеджеров пакетов в Windows ещё достаточно сырая.

В таком случае скрипт покажет ошибку и продолжит работу со следующими пакетами.

## Итог

Этот скрипт не заменяет полноценный менеджер обновлений, но закрывает простую практическую задачу: быстро пройтись по доступным обновлениям WinGet и выбрать, что именно обновлять сейчас.

Для регулярного ручного обслуживания Windows-системы этого вполне достаточно.
