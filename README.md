# FocoLab

Empilhamento de foco (*focus stacking*) para fotografia científica de espécimes, em um único arquivo HTML que roda no navegador.

Feito para material de estereoscópio: séries de 8 a 20 fotografias do mesmo espécime em planos de foco sucessivos, combinadas em uma imagem com o objeto inteiro nítido.

**Versão 1.0 · Licença GPL-3.0-or-later**

---

## Instalação

Não há instalação. Baixe `index.html` e abra no navegador.

Todo o processamento acontece no seu computador. Nenhuma imagem é enviada a servidor algum. O arquivo funciona offline, inclusive aberto diretamente do disco (`file://`).

Requisitos: um navegador com suporte a Web Workers, OffscreenCanvas e `createImageBitmap` — Chrome, Edge e derivados recentes. Firefox e Safari funcionam com limitações em alguns recursos.

---

## O que faz

- **Quatro métodos de fusão**, descritos abaixo.
- **Sete receitas** pré-configuradas, nomeadas pelo resultado que produzem, cada uma declarando o que prioriza e o que sacrifica.
- **Comparação de até cinco receitas** em prévia, para escolher pelo olho e não pela teoria.
- **Alinhamento opcional** entre planos, com correção de translação, escala e rotação.
- **Acabamento reversível**: realce de nitidez multiescala e esticamento de níveis, reaplicáveis sem refazer o empilhamento.
- **Realce de cor determinístico**: ponto branco automático, saturação seletiva e supressão de franja cromática.
- **Leitura de TIFF** sem dependências externas (sem compressão, LZW, Deflate, PackBits; 8 e 16 bits; tiras e blocos; little e big endian).
- **Processamento paralelo** em vários Workers, com particionamento por faixas.
- **Mapa de camadas e mapa de confiança** exportáveis como material suplementar.

---

## Os quatro métodos de fusão

| Método | Como decide | Indicado para |
|---|---|---|
| **Natural** | Monta um mapa de camadas, regulariza-o e usa uma camada por região, com transições suavizadas por filtro guiado. | Cutícula metálica, onde o reflexo especular se desloca entre planos. |
| **Pirâmide ponderada** | Decompõe cada quadro em escalas e combina os detalhes de todas as camadas com pesos por escala. | Uso geral. |
| **Pirâmide máxima** | Em cada escala, vence o quadro de maior contraste, sem média. | Cerdas, pontuações, esculturação da cutícula. |
| **Interpolado** | Usa o mapa de profundidade em resolução sub-quadro e interpola entre os dois planos vizinhos. | Superfícies lisas e contínuas, com passos regulares entre planos. |

---

## Créditos ao CombineZP

O FocoLab foi desenvolvido com o [CombineZP](https://github.com/Vincentdecursay/CombineZP) como **referência funcional e conceitual**. O CombineZP é obra de Alan Hadley, distribuído sob GPL, e continua sendo a referência prática para empilhamento de foco em material entomológico.

Ao longo do desenvolvimento, o código-fonte do CombineZP foi lido para entender decisões de projeto que não estão documentadas em outro lugar. **Nenhuma linha de código foi copiada** — o FocoLab é escrito do zero em JavaScript, enquanto o CombineZP é C++ para Windows. O que foi aproveitado está listado abaixo, com a origem exata.

### O que adotamos

**Ganhos por nível durante a reconstrução da pirâmide** (`Pyramid.cpp`, função `pascend`)
O CombineZP não aplica realce de nitidez ao final: ele reforça o detalhe *dentro* da reconstrução, nível a nível, com três parâmetros que o macro "Pyramid Maximum Contrast" fixa em 125, 107 e 93:

- `pforeground`: reforço do detalhe, aplicado apenas nas três escalas mais finas, segundo `contraste = 100 + (pforeground − 100) × (12 − nível) / 12`;
- `pbackground`: atenuação da base, apenas nas escalas mais grosseiras;
- `pbrightness`: ganho final de luminosidade, que compensa a atenuação da base.

O FocoLab implementa a mesma estrutura, com os três controles expostos ao usuário e neutros por padrão. É um realce **multiescala**, estruturalmente mais limpo que uma máscara de nitidez de raio único.

**Esticamento de níveis por histograma combinado** (`Special1.cpp`, função `contrast`)
O CombineZP monta um histograma único somando os três canais, encontra o ponto preto e o branco, e aplica o mesmo ganho linear a R, G e B. Além do contraste, isso amplia a distância entre canais, o que aparece como cor mais viva. O FocoLab usa o mesmo procedimento, com proteção adicional das altas luzes — necessária porque fundo de papel branco satura facilmente.

**A estrutura em macros**
A descoberta mais útil da leitura do código foi que o macro "Do Stack" do CombineZP tem **quatorze passos**, e apenas o primeiro é fusão. Os demais são regularização do mapa de profundidade (remoção de ilhas, preenchimento de lacunas, filtragem passa-baixa), reconstrução por interpolação, realce e ajuste de contraste. Essa observação orientou a arquitetura do FocoLab, que separa fusão, acabamento e cor em etapas independentes e reaplicáveis.

**A regularização do mapa de camadas**
O CombineZP nunca usa a seleção crua: ele remove ilhas, preenche lacunas e filtra o mapa antes de compor. O FocoLab adota a mesma ideia, com implementação própria.

**Interpolação entre planos vizinhos** (`Display.cpp`, `StackInterpolate`)
O CombineZP mantém um mapa de profundidade com resolução sub-quadro — índice inteiro nos bits altos, fração no byte baixo — e reconstrói interpolando entre os dois quadros que cercam cada pixel, em vez de somar todos. O método **Interpolado** do FocoLab segue esse princípio.

**Decisão em um canal compartilhado** (`Pyramid.cpp`, `pinw` e `pwa3`)
No CombineZP a escolha do quadro vencedor é feita uma única vez sobre um canal derivado de (R+G+B)/3, e os três canais de cor apenas copiam o coeficiente do vencedor. Testamos a alternativa — decidir por canal — e ela produz estriamento e franjas de cor. A abordagem do CombineZP está correta e é a que o FocoLab usa.

### O que testamos e não adotamos

Por honestidade técnica, vale registrar o que foi experimentado e descartado:

- **Porte direto do `PyramidMax`** (seleção por máximo absoluto do coeficiente laplaciano): produziu estriamento severo sobre cutícula metálica, com 12% a 23% de *overshoot* contra 0,41% do método atual. A causa é que o reflexo especular se desloca entre planos focais, e a seleção por máximo duro o persegue. Descartado.
- **Filtro passa-alta em DFT** (`Dft.cpp`, `CDft::High`, com largura 1,0 e deslocamento 0,75): atenua o DC para 0,617, o que escurece a imagem e estoura as sombras contra fundo branco. Uma máscara de nitidez comum alcança o mesmo resultado sem esse efeito e sem exigir FFT no navegador.
- **Aritmética inteira com valores escalados por 16**: herança de 2007. Em JavaScript moderno, `Float32Array` não é o gargalo.

### Nomenclatura

Os nomes dos métodos do FocoLab **não indicam equivalência** com os macros do CombineZP. "Pirâmide máxima" não é o "Pyramid Maximum Contrast"; é um método próprio, inspirado na mesma família de ideias.

---

## Decisões de projeto medidas

O desenvolvimento foi guiado por medições em séries reais, não por estimativa. Alguns resultados que orientaram o código:

**Três níveis de pirâmide, não cinco.** Passar de 3 para 7 níveis não altera a nitidez (variação de 0,5%) e multiplica o *overshoot* por 8.

**Detecção de foco multiescala.** Em escala única, a textura do papel de fundo pontua mais que uma antena escura e fina, e a antena herda o plano do fundo. Medido em uma série de 15 imagens: com passo de 1 px o vencedor sobre a antena era o quadro em que ela está desfocada; com passo de 4 a 16 px passa a ser o quadro correto.

**Alinhamento desligado por padrão.** Correlação de fase em resolução plena não encontrou deslocamento mensurável nas séries testadas, e cada correção custa uma reamostragem que consome de 8% a 17% da nitidez.

**Cache de bitmaps por faixa.** Cada processo paralelo guarda apenas a faixa horizontal que vai processar. Sem isso, uma série de 15 imagens de 8,3 MP exigia 2524 decodificações; com a faixa, 60. O tempo caiu de 227 s para 96 s.

---

## Licença

Copyright © 2026 Rodrigo Pereira — FFCLRP/USP

Este programa é software livre: você pode redistribuí-lo e/ou modificá-lo sob os termos da Licença Pública Geral GNU, publicada pela Free Software Foundation, na versão 3 da Licença ou, a seu critério, qualquer versão posterior.

Este programa é distribuído na expectativa de ser útil, mas SEM NENHUMA GARANTIA; nem mesmo a garantia implícita de COMERCIALIZAÇÃO ou ADEQUAÇÃO A UMA FINALIDADE ESPECÍFICA. Consulte a [Licença Pública Geral GNU](https://www.gnu.org/licenses/gpl-3.0.html) para mais detalhes.

Por ser um arquivo HTML único e não minificado, **o arquivo distribuído é simultaneamente o programa e seu código-fonte completo** — a exigência da GPL de disponibilizar a fonte é satisfeita pela própria cópia recebida.

O CombineZP, consultado como referência, é obra de Alan Hadley e também distribuído sob GPL.

---

## Como citar

> Pereira, R. (2026). *FocoLab: empilhamento de foco para fotografia científica de espécimes* (versão 1.0) [software]. FFCLRP/USP. Licença GPL-3.0.
