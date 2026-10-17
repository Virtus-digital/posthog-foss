# Virtus-digital/posthog-foss

[PostHog/posthog-foss](https://github.com/PostHog/posthog-foss) deposunun
(PostHog'un ee/ kodu çıkarılmış, MIT lisanslı aynası) Virtus fork'u. Tek işi,
fra kümesine dağıtılan PostHog uygulama imajlarını **bu FOSS ağacından**
derlemek:

| İmaj | Dockerfile | Kullanan servisler (hobby compose adlarıyla) |
|---|---|---|
| `ghcr.io/virtus-digital/posthog-foss:<commit>` | `Dockerfile` (yamalı, aşağıda) | web, worker, temporal-django-worker, asyncmigrationscheck |
| `ghcr.io/virtus-digital/posthog-foss-node:<commit>` | `Dockerfile.node` | plugins, ingestion-*, recording-api |

Dağıtım `Virtus-digital/deploy` → `platform-fra/posthog/` altında (henüz yok).
Rust servisleri (capture, feature-flags, property-defs-rs …) bu depodan
**derlenmiyor**; upstream'in `ghcr.io/posthog/posthog/*` imajları digest'le
sabitleniyor. Gerekçe K8s-structure'daki karar kaydında.

Son güncelleme: 2026-10-17

## Bu dalda ne var, ne yok

Varsayılan dal `virtus` upstream koduyla **ortak geçmiş taşımıyor**. İçinde
yalnız bu dosya ve `.github/workflows/imaj.yml` var. Upstream kodu `master`
dalında duruyor ama derleme onu **kullanmıyor** (aşağıda *Güncelleme yordamı*).

| Ne | Neden |
|---|---|
| `master` dalı duruyor (paperclip'ten farklı) | Upstream FOSS aynası `.github/workflows` taşımıyor (2026-10-16'da ölçüldü: `.github/` var, `workflows/` yok), yani `master`'da tetiklenecek bir upstream işi yok. Dal, derlenecek commit'lerin bu fork'ta bulunması için gerekli. |
| Varsayılan dal upstream dosyası taşımıyor | `workflow_dispatch` yalnız varsayılan daldaki workflow için açılıyor; zamanlanmış upstream işleri de yalnız varsayılan daldan koşar. |
| Sürüm etiketi yok, kimlik commit | PostHog sürüm etiketi yayımlamıyor (upstream `bin/deploy-hobby`: *"PostHog don't create tagged releases anymore"*). İmaj etiketi derlenen commit'in sha'sı. |

## Derleme yaması: `ee/`

FOSS ağacında `ee/` dizini yok, ama upstream `Dockerfile` onu üç yerde
istiyor (iki `COPY ... ee ee/` ve `compileall` listesi). Olduğu gibi
derlenmez. `imaj.yml`'in *"Dockerfile'i yamala"* adımı bu üç yeri siliyor ve
her birinin **tam bir kez** eşleşmesini şart koşuyor: upstream Dockerfile
değişirse adım kırmızı yanar, yama yanlış yere sessizce uygulanmaz. Yamanın
diff'i her koşunun özetine yazılıyor. Çalışma zamanında `ee/`'nin yokluğu
beklenen durum: `posthog/settings/ee.py` `ImportError`'da `EE_AVAILABLE =
False` bırakıyor.

## Derleme nerede koşuyor

Virtus ARC'de, kendi grubunda (`posthog-imaj`) ve kendi scale set'inde
(`Virtus-digital/deploy` → `platform-fra/arc-runners-posthog/`). Bu depo
public olduğu için org'un ana ARC grubu (`Default`) onu çalıştırmıyor.
Grup yalnız `.github/workflows/imaj.yml@refs/heads/virtus`'a açık; dosyanın
adı ya da dalı değişirse grubun `selected_workflows` listesi de
güncellenmeli:

```sh
gh api orgs/Virtus-digital/actions/runner-groups --jq '.runner_groups[] | select(.name=="posthog-imaj") | .selected_workflows'
```

Derlemeyi yalnız `imaj.yml`'in `if:` listesindeki kullanıcı başlatabilir;
başkası tetiklerse iş `skipped` olur, runner açılmaz.

## Güncelleme yordamı

⚠️ **2026-10-17'de değişti.** İlk yordam *"fork'u senkronla, master'dan
commit seç"* diyordu ve kaynak kapısı commit'in upstream master
**geçmişinde** olmasını istiyordu. İlk derleme (koşu 39028229557) bu kapıda
düştü ve sebebi ölçüldü: FOSS aynasının master'ı upstream
PostHog/posthog'un ham geçmişi + en üstte **tek** bir *"Sync and remove all
non-FOSS parts"* commit'i, ve her senkronda o commit **yeniden yazılıyor**.
Önceki senkronun commit'i (`8f87779`) yeni master'ın geçmişinde değil
(compare: `diverged`, ahead_by 100, behind_by 1). Geçmişin geri kalanı
`ee/`'li ham upstream commit'leri. Yani derlenebilecek tek FOSS ağacı
**o anki uç**.

1. Upstream ucunu oku:

   ```sh
   COMMIT=$(git ls-remote https://github.com/PostHog/posthog-foss refs/heads/master | cut -f1)
   ```

2. Derlemeyi başlat:

   ```sh
   gh workflow run imaj.yml -R Virtus-digital/posthog-foss --ref virtus -f commit="$COMMIT"
   ```

3. Koşunun özetindeki iki digest'i deploy'daki üretece yaz
   (`platform-fra/posthog/uret/README.md` → *Sürüm yükseltme*).

Kaynak kapısı derlemeden önce commit'in `PostHog/posthog-foss` master'ının
**ucu** olduğunu `git ls-remote` ile ölçüyor ve ağacı **upstream'den**
checkout ediyor; fork'un `master`'ı derlemede kullanılmıyor (Sync fork
gerekmez). Upstream derleme sırasında yeniden senkronlarsa kapı kırmızı
yanar, yeni uçla tekrar tetiklenir. Checkout'tan sonra `ee/`'nin yokluğu
iki imaj için de ayrıca ölçülüyor. Aynı etiket ikinci kez derlenmez.
