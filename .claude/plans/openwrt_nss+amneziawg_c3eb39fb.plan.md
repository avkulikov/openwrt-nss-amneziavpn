---
name: OpenWrt NSS+AmneziaWG
overview: "Репозиторий станет «builder assets» как в Qualcommax_NSS_Builder: GitHub Actions собирает OpenWrt (qosmio/openwrt-ipq main-nss) для Xiaomi AX3600 с NSS, добавляет AmneziaWG из amneziawg-openwrt, публикует релиз с образами и .apk пакетами."
todos:
  - id: bd-init
    content: Инициализировать beads (bd prime/onboard) и завести issue на задачу сборки AX3600 NSS + AmneziaWG + .apk.
    status: pending
  - id: scaffold-builder-assets
    content: Создать структуру builder-assets (workflow, ax3600.config, files overlay, docs, patches directories) на базе Qualcommax_NSS_Builder.
    status: pending
    dependencies:
      - bd-init
  - id: workflow-amneziawg-feed
    content: "Доработать .github/workflows/build.yaml: checkout amneziawg feed по SHA, подключение src-link, учет amnezia SHA в check_updates, публикация .apk в релиз."
    status: pending
    dependencies:
      - scaffold-builder-assets
  - id: config-amneziawg
    content: "Обновить ax3600.config: включить kmod-amneziawg/amneziawg-tools/luci-proto-amneziawg (и опционально qrencode)."
    status: pending
    dependencies:
      - scaffold-builder-assets
  - id: compat-patches
    content: "Подготовить patches/amneziawg/*: версионирование luci, фиксы совместимости под kernel main-nss (ребейз патча kmod при необходимости)."
    status: pending
    dependencies:
      - workflow-amneziawg-feed
      - config-amneziawg
  - id: docs-and-release
    content: "Обновить README/COMPATIBILITY: как собирать, где лежат образы и .apk, как прошивать AX3600 и включать AmneziaWG."
    status: pending
    dependencies:
      - compat-patches
---

# План: OpenWrt (AX3600) + NSS + AmneziaWG (.apk) + GitHub Releases

## Контекст и источники

- Базовый пример сборки/релизов: [Qualcommax_NSS_Builder](https://github.com/JuliusBairaktaris/Qualcommax_NSS_Builder/tree/main)
- Исходники пакета: [amneziawg-openwrt](https://github.com/amnezia-vpn/amneziawg-openwrt)
- Инструкции/референсы: [yury-sannikov/awg-openwrt](https://github.com/yury-sannikov/awg-openwrt), [Slava-Shchipunov/awg-openwrt](https://github.com/Slava-Shchipunov/awg-openwrt?tab=readme-ov-file)
- Устройство/таргет: [OpenWrt Xiaomi AX3600](https://openwrt.org/toh/xiaomi/ax3600)
- Про apk в OpenWrt: [OpenWrt apk](https://openwrt.org/docs/guide-user/additional-software/apk)
- Доп. инструкция по AWG: `https://m-docs-3w5hsuiikq-ez.a.run.app/documentation/instructions/openwrt-os-awg/`

## 0) Тасктрекинг (bd)

- Запустить `bd prime --stealth`, затем `bd onboard`.
- Создать/взять issue на задачу сборки (NSS + AmneziaWG + .apk + релизы) и пометить `in_progress`.

## 1) Скелет репозитория (builder-assets как у Qualcommax)

Добавим файлы/директории:

- [`.github/workflows/build.yaml`](.github/workflows/build.yaml): основной workflow сборки и релиза (на базе `Qualcommax_NSS_Builder/.github/workflows/build.yaml`).
- [`ax3600.config`](ax3600.config): конфиг OpenWrt (на базе `Qualcommax_NSS_Builder/ax3600.config`) + включение AmneziaWG.
- [`files/etc/uci-defaults/999-QOL_config`](files/etc/uci-defaults/999-QOL_config): как в примере (можно без изменений).
- [`files/etc/uci-defaults/999-sqm-settings`](files/etc/uci-defaults/999-sqm-settings): как в примере (можно без изменений).
- [`patches/openwrt/`](patches/openwrt/): опциональные патчи к `qosmio/openwrt-ipq`.
- [`patches/amneziawg/`](patches/amneziawg/): патчи к фиду `amneziawg-openwrt` для совместимости с OpenWrt 25+/apk и текущим kernel в `main-nss`.
- [`README.md`](README.md): как запускать сборку/где артефакты/как прошивать AX3600.
- [`COMPATIBILITY.md`](COMPATIBILITY.md): что именно поправлено в AmneziaWG под OpenWrt 25+/ядро.

## 2) GitHub Actions: сборка OpenWrt+NSS+AmneziaWG и релизы

В [`.github/workflows/build.yaml`](.github/workflows/build.yaml):

- **База как в Qualcommax**: `REMOTE_REPOSITORY=qosmio/openwrt-ipq`, `REMOTE_BRANCH=main-nss`, `NSS_PACKAGES_REPOSITORY=qosmio/nss-packages`, `NSS_PACKAGES_REPOSITORY_BRANCH=NSS-12.5-K6.x` (как в исходном workflow).
- **Добавим AmneziaWG как ещё один upstream**, чтобы билд триггерился при его изменениях тоже:
- env: `AMNEZIAWG_REPOSITORY=amnezia-vpn/amneziawg-openwrt`, `AMNEZIAWG_BRANCH=master`.
- job `check_updates`: получить `amnezia_sha` через `gh api` и учитывать его в `build_required` (аналогично `main_sha`/`nss_sha`).
- **Checkout AmneziaWG feed на конкретный SHA**:
- `actions/checkout` в путь `amneziawg_feed` с `repository: ${{ env.AMNEZIAWG_REPOSITORY }}` и `ref: ${{ env.AMNEZIA_SHA }}`.
- применить патчи из `builder_repo/patches/amneziawg/*.patch` (если есть) прямо к `amneziawg_feed`.
- **Подключить feed в OpenWrt**:
- в `openwrt/feeds.conf`: добавить `src-link amneziawg ../amneziawg_feed`.
- `./scripts/feeds update amneziawg && ./scripts/feeds install -a -p amneziawg`.
- **Конфиг и сборка**: как в примере — `cp ax3600.config .config`, `make defconfig`, `make download`, `make -j$(nproc)`.
- **Артефакты и релиз**:
- как минимум публиковать sysupgrade/factory файлы из `openwrt/bin/targets/qualcommax/ipq807x/`.
- дополнительно: найти и приложить **.apk пакеты AmneziaWG** (т.к. вы выбрали “и в образ, и .apk в релиз”):
- после сборки собрать список `*.apk` по шаблону `*amneziawg*.apk` (включая `kmod-amneziawg`, `amneziawg-tools`, `luci-proto-amneziawg`) из `openwrt/bin/**`.
- `gh release upload` этими файлами в созданный релиз.
- добавить в release notes 3 SHA: `Main Commit`, `NSS Commit`, `AmneziaWG Commit`.

## 3) Изменения в `ax3600.config` (предустановка AmneziaWG)

В [`ax3600.config`](ax3600.config) добавим:

- `CONFIG_PACKAGE_kmod-amneziawg=y`
- `CONFIG_PACKAGE_amneziawg-tools=y`
- `CONFIG_PACKAGE_luci-proto-amneziawg=y`
- (опционально для LuCI UX) `CONFIG_PACKAGE_qrencode=y`

## 4) Совместимость AmneziaWG с OpenWrt 25+/apk (и текущим kernel main-nss)

Вы выбрали **strict_upstream**, поэтому план такой:

- **Сначала пробуем собирать `amneziawg-openwrt` как есть**.
- Если сборка падает:
- **kmod**: ребейзим/правим патч `kmod-amneziawg/files/000-initial-amneziawg.patch` под актуальные `$(LINUX_DIR)/drivers/net/wireguard/*` в `qosmio/openwrt-ipq@main-nss`.
- **luci-proto**: привести версионирование к “классическому” OpenWrt стилю (уменьшить риск проблем с apk): заменить `PKG_VERSION:=0.0.1-1` на `PKG_VERSION:=0.0.1` и добавить `PKG_RELEASE:=1` (как патч в [`patches/amneziawg/`](patches/amneziawg/)).
- фиксируем любые изменения зависимостей/путей в `amneziawg-tools`/`luci-proto` в виде патчей в [`patches/amneziawg/`](patches/amneziawg/).

## 5) Проверки результата (в CI)

В workflow добавим простые проверки:

- Убедиться, что собран профиль AX3600: наличие sysupgrade/factory артефактов в `openwrt/bin/targets/qualcommax/ipq807x/`.
- Убедиться, что **.apk артефакты AmneziaWG существуют**: `find openwrt/bin -name '*.apk' | grep -E 'amneziawg|luci-proto-amneziawg|kmod-amneziawg'`.

## 6) Завершение сессии (обязательные шаги проекта)

- После внесения изменений: прогон сборки (минимум через GitHub Actions), обновить/закрыть issue в bd.
- Выполнить «Landing the plane»: `git pull --rebase`, `bd sync`, `git push`, убедиться `git status` показывает up-to-date.