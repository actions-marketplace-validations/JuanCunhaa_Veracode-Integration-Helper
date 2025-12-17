# Bantuu Veracode Baseline

Action para rodar o **Veracode Pipeline Scan** integrado com baseline vindo do **Bantuu**.

- Se **não houver baseline** no Bantuu para o repositório (`org/repo`):
  - Roda o Pipeline Scan sem baseline;
  - Gera `results.json`;
  - Envia o `results.json` para o Bantuu para ser usado como baseline futuro.
- Se **já existir baseline**:
  - Baixa o baseline do Bantuu;
  - Cria `baseline.json` local;
  - Roda o Pipeline Scan usando `baseline.json` como baseline.

---

## Requisitos

- Repositório usando GitHub Actions;
- Secrets configurados no repositório:
  - `BANTUU_API_KEY`
  - `VERACODE_API_ID`
  - `VERACODE_API_KEY`
- Artefato compactado para envio ao Veracode (ex.: `app.zip`).

---

## Exemplo de workflow (`.github/workflows/veracode-bantuu.yml`)

```yaml
name: Security Scan (Veracode + Bantuu)

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  veracode-baseline:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build and package application
        run: |
          # Exemplo: gere o artefato para o Veracode, por exemplo app.zip
          zip -r app.zip .

      - name: Bantuu Veracode Baseline
        id: bantuu_veracode_baseline
        uses: Afrikatec-JuanCunhaa/bantuu-baseline@v1
        with:
          bantuu_api_key: ${{ secrets.BANTUU_API_KEY }}
          # Opcional: sobrescrever URL base do Bantuu (por exemplo, ambiente de dev/homolog)
          # bantuu_base_url: 'https://bantuu-dev.example.com'
          veracode_api_id: ${{ secrets.VERACODE_API_ID }}
          veracode_api_key: ${{ secrets.VERACODE_API_KEY }}
          scan_file: app.zip
          policy_fail: 'false'
          # Opcional: só será aplicado quando houver baseline
          # fail_on_severity: 'Very High, High'
```

---

## Inputs

- `bantuu_api_key` (obrigatório)  
  API Key usada no header `X-API-Key` do Bantuu (GET baseline e POST upload).

- `bantuu_base_url` (opcional, default: `"https://bantuu.io"`)  
  Base URL do Bantuu. Use para apontar para ambientes de desenvolvimento/homologação quando necessário.

- `veracode_api_id` (obrigatório)  
  ID da API do Veracode.

- `veracode_api_key` (obrigatório)  
  Key da API do Veracode.

- `scan_file` (obrigatório)  
  Arquivo compactado enviado ao Pipeline Scan (ex.: `app.zip`).

- `policy_fail` (opcional, default: `"false"`)  
  Define se o scan deve quebrar o build por policy (`true`/`false`).

- `fail_on_severity` (opcional)  
  Severidades que fazem o scan falhar **quando já existe baseline** (ex.: `"Very High, High"`).

---

## Outputs

- `has_baseline`  
  Indica se já existia baseline no Bantuu (`true`/`false`).

- `pipeline_status`  
  Indica se o scan rodou com baseline ou sem baseline e fez upload:
  - `scan_completed_with_baseline`
  - `scan_completed_without_baseline_and_uploaded`

- `repository_full_name`  
  `org/repo` usado para consultar e enviar baseline ao Bantuu.

---

## Usando os outputs no workflow

Exemplo de como reagir ao resultado do scan e ao uso de baseline:

```yaml
      - name: Verificar se havia baseline no Bantuu
        run: |
          echo "Has baseline: ${{ steps.bantuu_veracode_baseline.outputs.has_baseline }}"
          echo "Pipeline status: ${{ steps.bantuu_veracode_baseline.outputs.pipeline_status }}"

      - name: Ação quando não havia baseline
        if: ${{ steps.bantuu_veracode_baseline.outputs.pipeline_status == 'scan_completed_without_baseline_and_uploaded' }}
        run: echo "Primeiro scan concluído, baseline criado no Bantuu."
```

---

## Notas e limitações

- A Action foi projetada para rodar em `ubuntu-latest` (usa `apt-get` e `sudo` para instalar `jq`).
- Requer acesso de rede até a URL configurada em `bantuu_base_url` (por padrão `https://bantuu.io`) a partir do runner.
- A URL base padrão do Bantuu é `https://bantuu.io`, mas pode ser sobrescrita via input `bantuu_base_url` (por exemplo, para ambientes de dev/staging).
- É responsabilidade do workflow gerar o arquivo indicado em `scan_file` antes da chamada.
- O scan é executado via `veracode/Veracode-pipeline-scan-action@v1.0.20`.

