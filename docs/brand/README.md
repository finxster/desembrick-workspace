# Desembrick: guia de marca (v1.0)

Resumo das regras para quem mexe no código. O guia completo é o PDF de 11 páginas (`desembrick-identidade-visual.pdf`); os valores vivem em `tokens.css`.

## Arquivos

| Arquivo | Uso |
|---|---|
| `tokens.css` | Cores (claro e escuro), tipografia, raios e código de cor dos sacos |
| `svg/logo*.svg` | Assinatura: símbolo + wordmark (texto já convertido em curvas) |
| `svg/symbol*.svg` | Símbolo isolado |
| `svg/wordmark*.svg` | Wordmark isolado |
| `svg/app-icon*.svg` | Ícone de app, sangrado até a borda |
| `svg/favicon.svg` | Favicon desenhado para 16 px |

Sufixos: sem sufixo = fundo claro; `-negative` = fundo escuro; `-mono` e `-mono-negative` = uma cor só.

## Logo

- Respiro mínimo em todos os lados: o diâmetro de um pino do símbolo.
- Abaixo de 120 px de largura (30 mm impresso), use só o símbolo. Símbolo mínimo: 16 px (5 mm).
- Não esticar, girar, trocar as cores dos pinos, aplicar sombra ou contorno, nem usar sobre fundo colorido sem contraste.
- Não combinar com nomes, logotipos ou personagens de fabricantes de blocos. O tijolinho é sempre genérico.

## Cores

- Tinta e papel ocupam a maior parte de qualquer tela ou página. Azul, vermelho e amarelo são acentos.
- Sacos seguem o ciclo azul, vermelho, amarelo (`.db-bag` em `tokens.css`). A cor nunca é a única pista: o número aparece sempre.
- Use só os pares texto/fundo dos tokens `--db-color-on-*`. No tema claro: branco sobre azul 4,7:1, branco sobre vermelho 4,7:1, tinta sobre amarelo 10,7:1, tinta sobre papel 15,8:1, apoio sobre papel 5,9:1.
- No tema escuro o texto sobre as cores da marca é o fundo escuro (#141318): sobre azul 7,2:1, sobre vermelho 5,9:1, sobre amarelo 11,6:1; texto sobre fundo 16,6:1, apoio sobre fundo 8,9:1.

## Tipografia

- Space Grotesk Bold: títulos, wordmark e número do saco, com espaçamento negativo.
- IBM Plex Sans Regular e SemiBold: texto e interface.
- IBM Plex Mono: números, quantidades, códigos e legendas em caixa-alta.
- As três são gratuitas (SIL OFL). Pesos usados: Space Grotesk 400, 500, 700; Plex Sans 400, 500, 600; Plex Mono 400, 500, 600.

## Voz e tom

- Português do Brasil primeiro, inglês ao lado.
- Frases curtas, verbo no começo: diga o que aconteceu e o próximo passo.
- Humor leve em sucessos e telas vazias; em erros, só clareza. Nunca infantil.
- Vocabulário: "blocos de montar", "tijolinho", "set", "saco". Nunca nomes de fabricantes ou de linhas de produto.

| Situação | Português | English |
|---|---|---|
| Botão | Gerar manual reverso | Generate reverse manual |
| Tela vazia | Nenhum set por aqui ainda. Envie um manual em PDF para começar. | No sets here yet. Upload a PDF manual to get started. |
| Sucesso | Pronto! 12 sacos, cada peça no seu lugar. | Done! 12 bags, every piece in its place. |
| Erro | Não consegui ler esta página do manual. Tente outro arquivo. | I couldn't read this page of the manual. Try another file. |

## Página do manual reverso

- Número do saco: bloco na cor do saco, Space Grotesk Bold, maior elemento da página.
- Título: nome do set, posição do saco ("Saco 03 de 12") e passos cobertos.
- Grade de peças: seis colunas, conteúdo centralizado (quantidade, desenho, nome, cor), fundo branco e linhas finas para imprimir bem.
