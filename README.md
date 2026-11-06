# Gerber para CopperCAM

Converte o Gerber exportado pelo **EasyEDA Pro** para um formato que o **CopperCAM** lê sem erro, e confere a placa do jeito que a fresa vai cortar.

Tudo roda no navegador: os arquivos da placa não são enviados para lugar nenhum. A página funciona online (GitHub Pages) ou offline, abrindo o `index.html` com dois cliques.

## Como usar

1. No EasyEDA Pro: **Export → PCB Fabrication File (Gerber)**.
2. Abra a página e solte o `.zip` exportado na caixa (ou clique nela para escolher o arquivo).
3. Leia o veredito e a lista "O que conferir". Cada aviso tem a coordenada em mm, a partir do canto inferior esquerdo da placa, e um número marcado na prévia.
4. Clique em **Baixar pacote para o CopperCAM**.
5. No CopperCAM, abra o `_Bottom.gbr`, depois o `_Contorno.gbr` e o `_Furos.drl`. Se ele perguntar se quer reduzir furos maiores que os pads, responda **Não**.

## O que a conversão faz

- Regrava o cobre Bottom e o contorno em **polegada, formato 2.4**, sem comentários nem atributos X2.
- **Renumera as aberturas a partir de D20** e deixa D10 e D11 sem uso. O CopperCAM lê errado os pads que caem no código D11 (eles aparecem sem cobre e ele avisa "Some holes are bigger than their pads").
- Junta os arquivos de furação (PTH, vias, NPTH), remove furos repetidos e unifica ferramentas de mesmo diâmetro. O arquivo de vias do EasyEDA repete furos que já estão no PTH.
- Troca furos oblongos (slots) por furos redondos.
- Não espelha nada: o CopperCAM faz isso na hora de fresar.

## O que ela confere

| Checagem | Critério |
|---|---|
| Folga entre cobres | abaixo de 0,3 mm: corrigir; de 0,3 a 0,45 mm: atenção |
| Cobre perto da borda | menos de 1 mm do contorno |
| Anel de cobre dos furos | menos de 0,3 mm, ou furo sem cobre em volta |
| Cobre solto | ilha de cobre sem nenhum furo |
| Jumpers | ligações na camada Top (viram fio numa placa de face simples) |

## Limitações

- Feita e testada com Gerbers do **EasyEDA Pro**. Outros programas podem funcionar, mas não foram testados.
- Pads com forma especial (macro de abertura, como retângulo arredondado) não são suportados: troque por pad redondo, retangular ou oval.
- A checagem de borda considera a placa retangular.
- Considera placa de face simples, com o cobre no Bottom.

## Publicar no GitHub Pages

1. Crie um repositório público no GitHub (por exemplo, `gerber-coppercam`).
2. Envie o `index.html` e este `README.md` para a raiz do repositório (pelo botão **Add file → Upload files** já funciona).
3. No repositório: **Settings → Pages → Build and deployment → Source: Deploy from a branch**, escolha a branch `main` e a pasta `/ (root)` e salve.
4. Em um ou dois minutos o site fica em `https://SEU-USUARIO.github.io/gerber-coppercam/`.

Para atualizar, é só enviar o `index.html` novo para o repositório.

## Créditos

Usa a biblioteca [JSZip](https://stuk.github.io/jszip/) 3.10.1 (licença MIT ou GPLv3), embutida no `index.html`.
