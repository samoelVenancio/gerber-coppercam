# Gerber para CopperCAM

Converte o Gerber exportado pelo **EasyEDA Pro** para um formato que o **CopperCAM** lê sem erro, e confere a placa do jeito que a fresa vai cortar.

Tudo roda no navegador: os arquivos da placa não são enviados para lugar nenhum. A página funciona online (GitHub Pages) ou offline, abrindo o `index.html` com dois cliques.

## Como usar

1. No EasyEDA Pro: **Export → PCB Fabrication File (Gerber)**.
2. Abra a página e escolha o tipo de placa (**face simples** ou **dupla face**) e os parâmetros da sua ferramenta.
3. Solte o `.zip` exportado na caixa (ou clique nela para escolher o arquivo).
4. Leia o veredito e a lista "O que conferir":
   - clique num aviso para a prévia ir até o ponto;
   - clique num diâmetro da tabela de furação para ver onde ficam esses furos;
   - roda do mouse ou dois dedos dão zoom, e arrastar move a prévia.
5. Clique em **Baixar pacote para o CopperCAM**.
6. No CopperCAM, abra o `_Bottom.gbr` (e o `_Top.gbr`, se for dupla face), depois o `_Contorno.gbr` e o `_Furos.drl`. Se ele perguntar se quer reduzir furos maiores que os pads, responda **Não**.

## Parâmetros

| Parâmetro | Padrão | O que muda |
|---|---|---|
| Tipo de placa | Face simples | Na face simples, só o Bottom é conferido e as trilhas da Top contam como jumpers. Na dupla face, as duas camadas são conferidas e saem no pacote. |
| Largura de corte da isolação | 0,30 mm | Folga menor que isso: corrigir. Até 0,15 mm acima disso: atenção. |
| Distância mínima da borda | 1,0 mm | Cobre mais perto do contorno do que isso gera aviso. |
| Anel mínimo de cobre | 0,30 mm | Cobre em volta do furo menor que isso gera aviso. |
| Brocas disponíveis | (vazio) | Opcional. Cada furo usa a menor broca que não fique menor que ele, e o anel é conferido com essa broca. O arquivo de furação continua com os diâmetros do projeto. |

Os parâmetros ficam salvos no navegador de quem usa.

## O que a conversão faz

- Regrava o cobre e o contorno em **polegada, formato 2.4**, sem comentários nem atributos X2.
- **Renumera as aberturas a partir de D20** e deixa D10 e D11 sem uso. O CopperCAM lê errado os pads que caem no código D11 (eles aparecem sem cobre e ele avisa "Some holes are bigger than their pads").
- Junta os arquivos de furação (PTH, vias, NPTH), remove furos repetidos e unifica ferramentas de mesmo diâmetro. O arquivo de vias do EasyEDA repete furos que já estão no PTH.
- Troca furos oblongos (slots) por furos redondos.
- Não espelha nada: o CopperCAM faz isso na hora de fresar.

## Limitações

- Feita e testada com Gerbers do **EasyEDA Pro**.
- Pads com forma especial (macro de abertura, como retângulo arredondado) não são suportados: troque por pad redondo, retangular ou oval.
- A checagem de borda considera a placa retangular.

## Publicar no GitHub Pages

1. Envie o `index.html` para a raiz do repositório.
2. **Settings → Pages → Deploy from a branch**, branch `main`, pasta `/ (root)`.
3. O site fica em `https://SEU-USUARIO.github.io/NOME-DO-REPOSITORIO/`.

## Desenvolvimento

O `CLAUDE.md` descreve a estrutura do código, o que o CopperCAM exige e como validar mudanças.

## Créditos

Usa a biblioteca [JSZip](https://stuk.github.io/jszip/) 3.10.1 (licença MIT ou GPLv3), embutida no `index.html`.
