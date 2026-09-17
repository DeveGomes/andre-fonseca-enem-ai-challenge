# Design — Gabaritei AI

> Tokens e regras visuais. Se um valor não estiver aqui, ele não deve aparecer no código.
> Mockups das telas: canvas do projeto. Arquivos da marca: `docs/assets/`.

---

## 1. Princípios

Quatro regras que decidem qualquer dúvida de layout:

1. **A tela responde "o que eu faço agora" antes de "como estou indo".** Ação tem mais peso visual que métrica.
2. **O verde significa alguma coisa.** Ação, acerto e marca. Nunca decoração, nunca fundo de seção.
3. **Menos caixa.** Card só onde há ação ou dado agrupado. Caixa dentro de caixa é ruído.
4. **Estado vazio é tela, não ausência de tela.** Nada de `0%`, gráfico vazio ou `NaN`.

---

## 2. Cores

| Token | Hex | Onde usar |
|---|---|---|
| `bg` | `#0B0F14` | fundo da aplicação |
| `surface` | `#101720` | cards, painéis |
| `surface-2` | `#0D1219` | sidebar, barra inferior no mobile |
| `surface-3` | `#131C25` | item de menu ativo |
| `border` | `#1E2730` | borda de card |
| `border-soft` | `#1A222B` | divisórias e filetes |
| `ink` | `#E9EEF2` | texto principal |
| `ink-2` | `#93A1AD` | texto secundário |
| `ink-3` | `#7C8A95` | rótulos, legendas |
| `ink-off` | `#4E5A64` | item desabilitado |
| **`green`** | `#2FD08C` | **ação, acerto, marca** |
| `green-soft` | `#5FE3B0` | texto sobre fundo verde escuro |
| `green-bg` | `#0F2A22` | fundo do bloco de ação principal |
| `green-bg-2` | `#0E1C18` | faixa de simulado em andamento |
| `green-border` | `#235741` | borda dos blocos verdes |
| `green-ink` | `#062017` | texto sobre botão verde |
| `navy` | `#16364D` | marca, selo do avatar, numeradores |
| `track` | `#17202A` | trilho das barras do gráfico |

### Contraste — verificado

Todos os pares de texto passam de 4.5:1. `ink` em `bg` dá 16.4:1, `ink-3` dá 4.5:1 (use a partir de 12px), `green` em `bg` dá 9.6:1 e `green-ink` sobre `green` dá 8.6:1.

`border` e `border-soft` são **só** para filetes — nunca para texto.

---

## 3. Tipografia

**Manrope**, via Google Fonts, pesos 400 a 800. Uma família só.

| Uso | Tamanho | Peso | Tracking |
|---|---|---|---|
| Título de página | 26px (30px no vazio) | 800 | −0.02em |
| Número grande | 29px | 800 | −0.03em |
| Título de card de ação | 19px | 700 | −0.015em |
| Título de painel | 15px | 700 | normal |
| Texto | 13.5–14px | 400 | normal |
| Rótulo secundário | 12–13px | 400 | normal |
| Etiqueta caixa alta | 10px | 700 | 0.14em |

Números grandes sempre com tracking negativo — é o que faz o peso 800 não parecer pesado demais.

---

## 4. Forma e espaço

| Token | Valor |
|---|---|
| Raio de card e botão | 6–8px |
| Raio do selo da marca | 7px (no viewBox de 32) |
| Raio da barra de gráfico | 4px |
| Espaço entre seções | 18–22px |
| Padding de card | 22px 24px |
| Padding de botão | 11px 22px |
| Alvo de toque mínimo | 44px |
| Sidebar | 244px |
| Conteúdo (desktop) | padding 30px 34px |

Sem sombra. Profundidade vem de superfície + borda, não de blur.

---

## 5. Componentes

**Botão primário:** fundo `green`, texto `green-ink`, peso 700.
**Botão secundário:** transparente, borda `#2B3742`, texto `ink`, peso 600.
**Botão em bloco verde:** transparente, borda `#2F6A4F`, texto `green-soft`.
**Desabilitado:** borda `#222B34`, texto `ink-off`, `cursor: not-allowed` — e **sempre** com uma frase dizendo por quê.

**Card de ação principal:** fundo `green-bg`, borda `green-border`, etiqueta caixa alta em `green-soft`.
**Card neutro:** fundo `surface`, borda `border`.
**Card de estatística:** rótulo 12px, número 29px/800, e à direita um quadrado 34px raio 7px em `#16293A` com ícone `green-soft` 16px.

**Item de menu ativo:** fundo `surface-3`, texto `ink`, ícone `green`. Inativo: texto `ink-2`, ícone `ink-3`. **Sem barra colorida na lateral.**

**Ícones:** traço (stroke) de 1.8, `stroke-linecap="round"`, 18px na sidebar e 16px nos cards. Nunca emoji.

---

## 6. Gráficos

- **Barras (perfil de erros):** série única, então **uma cor só** (`green`) — barra colorida por categoria aqui seria arco-íris decorativo. Altura 8px, trilho `track`, raio 4px. Rótulo à esquerda em `ink`, valor à direita em `ink-2`.
- **Linha (evolução):** traço de 2.5px em `green`, grade recessiva em `border-soft`, marcador de 5.5px só no **último** ponto, com rótulo. Nunca número em cima de todo ponto.
- Toda `<svg>` de gráfico leva `role="img"` e um `aria-label` que descreve a tendência em uma frase.

---

## 7. Marca

Beca sobre um certo: o objetivo e o acerto no mesmo símbolo.

Arquivos: [`docs/assets/logo.svg`](assets/logo.svg) · [`docs/assets/favicon.svg`](assets/favicon.svg)

```svg
<svg viewBox="0 0 32 32" role="img" aria-label="Gabaritei AI">
  <rect x="0" y="0" width="32" height="32" rx="7" fill="#16364D"/>
  <path d="M16 5.5 L25.5 9.5 L16 13.5 L6.5 9.5 Z" fill="#E9EEF2"/>
  <path d="M9 19.5 L13.5 24 L23.5 13.8" stroke="#2FD08C" stroke-width="3.2"
        stroke-linecap="round" stroke-linejoin="round" fill="none"/>
</svg>
```

Regras: no favicon (20px) o traço do certo vai para **3.8**, senão some na redução. A assinatura é o selo + "Gabaritei" em 800 + "AI" em 700, 0.16em, verde. Sobre fundo claro o "AI" vira `#0B7A52`.

---

## 8. Tailwind

```js
// tailwind.config.js
export default {
  theme: {
    extend: {
      colors: {
        bg: '#0B0F14',
        surface: { DEFAULT: '#101720', 2: '#0D1219', 3: '#131C25' },
        line: { DEFAULT: '#1E2730', soft: '#1A222B' },
        ink: { DEFAULT: '#E9EEF2', 2: '#93A1AD', 3: '#7C8A95', off: '#4E5A64' },
        green: {
          DEFAULT: '#2FD08C', soft: '#5FE3B0', ink: '#062017',
          bg: '#0F2A22', bg2: '#0E1C18', border: '#235741',
        },
        navy: '#16364D',
        track: '#17202A',
      },
      fontFamily: { sans: ['Manrope', 'system-ui', 'sans-serif'] },
      borderRadius: { card: '8px', btn: '6px' },
    },
  },
}
```

---

## 9. Responsivo

Mobile-first. O desktop é adaptação.

| | Mobile (< 768px) | Desktop |
|---|---|---|
| Navegação | barra fixa embaixo, 4 itens | sidebar de 244px |
| Cards de resumo | 3 em linha, compactos | 4 em linha, com ícone |
| Ações | empilhadas, botão 100% | duas colunas |
| Gráfico de linha | oculto | visível |

O gráfico de linha some no celular de propósito: em 390px ele vira um rabisco ilegível. O que importa no celular é a ação e o perfil de erros.

---

## 10. Acessibilidade

- `<button>` e `<a href>` de verdade — nunca `onClick` em `div`.
- `aria-label` em todo botão que é só ícone.
- Identidade nunca só por cor: o item ativo do menu tem peso e fundo, não apenas cor.
- Alvos de toque de 44px no mobile.
- Foco visível — não remova o `outline` sem colocar outro no lugar.
