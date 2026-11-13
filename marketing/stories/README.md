# Stories do Instagram (lançamento)

Três stories em 1080×1920, postados em novembro de 2026.

| Arquivo | Conteúdo |
|---|---|
| `1-aviso` | Aviso: o próximo story é para quem tem blocos de montar |
| `2-desembrick` | Chamada com o logo e área para o sticker de link |
| `3-como-funciona` | Passo a passo em três etapas |

Cada `.png` é gerado a partir do `.html` de mesmo nome. Estilos comuns em
`base.css`; logos copiados de `docs/brand/svg/` (as fontes vêm do Google Fonts,
então é preciso internet).

## Gerar os PNGs de novo

Rode de dentro desta pasta:

```bash
for f in 1-aviso 2-desembrick 3-como-funciona; do "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu --hide-scrollbars --window-size=1080,1920 --virtual-time-budget=4000 --screenshot="$PWD/$f.png" "file://$PWD/$f.html"; done
```

## Regras de marca

Seguem `docs/brand/README.md`: "blocos de montar" nos títulos; LEGO® só para
dizer com o que funciona, sempre com ®, mais o aviso de marca registrada.
