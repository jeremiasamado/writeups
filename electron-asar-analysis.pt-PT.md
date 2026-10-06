# Análise Electron ASAR: Trust Boundaries e Validação no Cliente

[EN](./electron-asar-analysis.md)

<img src="https://readme-typing-svg.demolab.com?font=VT323&size=22&duration=2600&pause=1000&color=8A00C4&background=00000000&center=true&vCenter=true&width=700&height=36&lines=ELECTRON+%2F+ASAR+%2F+FLUXO+DE+ENTITLEMENT;MAPEAR+O+CLIENTE+ANTES+DE+CONFIAR+NELE" alt="Fluxo de entitlement Electron ASAR">

**Autor:** NE0SYNC  
**Data:** Setembro de 2026  
**Foco:** internals Electron, estrutura ASAR, fluxo de entitlement e trust boundaries  
**Ferramentas:** Detect It Easy, Python, tooling ASAR, Frida, IDA Pro

## Âmbito

Esta nota documenta uma análise de reverse engineering autorizada a uma
aplicação Electron. O alvo e os detalhes identificativos foram omitidos de
propósito. Não são distribuídos binários, credenciais ou instruções operacionais
de bypass.

## Hipótese inicial

A aplicação parecia combinar três camadas:

1. um frontend empacotado em ASAR;
2. um serviço Windows local envolvido na validação;
3. um backend remoto que devolvia dados de entitlement assinados.

A primeira pergunta não foi “onde está o switch premium?”. Foi:

> Onde decide a aplicação aquilo a que o utilizador tem acesso, e que parte
> dessa decisão é considerada fidedigna pelo cliente?

## Arquitectura observada

```text
Electron main process
        │
        ├── serviço local de validação
        │
        └── renderer process
                │
                └── resposta de entitlement assinada

backend remoto ─────────────────┘
```

O pacote era um archive ASAR. A lógica relevante estava dividida entre o main
process e os assets do renderer, por isso ler um único ficheiro daria uma visão
incompleta.

## Análise estática

A primeira passagem usou identificação do tipo de ficheiro e inspecção do
archive para perceber a estrutura da aplicação. A separação relevante era:

- `out/main/main.js` — orquestração e comunicação com o serviço local;
- bundle do renderer — apresentação e interpretação dos dados de entitlement;
- metadata do pacote — entrypoints e contexto de runtime.

O resultado importante foi arquitectural, não cosmético: o cliente continha
lógica suficiente para observar e influenciar a apresentação final do entitlement.

## Observações em runtime

A observação em runtime serviu para comparar o fluxo esperado com o que o
cliente realmente consumia:

```text
arranque
  ↓
main process pede validação
  ↓
serviço local comunica com o backend
  ↓
entitlement assinado chega ao renderer
  ↓
renderer selecciona um access level
```

O serviço local e a verificação de assinatura acrescentavam camadas, mas o
cliente continuava a participar na decisão final de acesso. A distinção é
importante: a integridade de uma resposta não torna autoritária uma decisão
controlada pelo cliente.

## Finding

**O cliente aplicava parte da fronteira de entitlement localmente.**

A aplicação tratava o estado do cliente como uma barreira relevante para
funcionalidade premium. Quem controla o processo cliente consegue inspeccionar
esse estado e alterar o caminho depois de a resposta ser recebida.

O finding não é “uma string pode ser alterada”. O finding é que a trust boundary
foi colocada demasiado perto do cliente.

## Impacto

Quando uma capacidade sensível é libertada apenas depois de uma decisão no
cliente, um cliente modificado pode apresentar ou invocar funcionalidade que
deveria ter sido autorizada pelo serviço. O impacto depende do que o backend
valida de forma independente:

- um gate apenas de UI limita o problema à apresentação;
- verificações server-side de entitlement protegem operações no backend;
- assets premium apenas locais continuam expostos a quem controla o pacote.

É importante distinguir estes casos, em vez de tratar qualquer flag visível no
cliente como tendo automaticamente o mesmo impacto.

## Remediação de engenharia

Um desenho mais forte mantém a decisão autoritativa no servidor:

1. validar o entitlement na fronteira do serviço;
2. emitir capability tokens curtos e associados ao audience correcto;
3. autorizar operações sensíveis em todos os caminhos relevantes do backend;
4. enviar ao cliente apenas os dados mínimos para a operação actual;
5. tratar verificações no renderer e no main process como UX e controlo de
   fluxo, nunca como a fronteira final de segurança.

Verificações de integridade do pacote podem detectar alterações, mas não
substituem autorização server-side.

## Limitações

Este write-up não inclui identificadores do alvo, binários, localizações exactas
de patches, hooks ou instruções que permitam contornar uma aplicação comercial.
O objectivo é mostrar o caminho de análise e a lição de desenho.

## Conclusão

As aplicações Electron são JavaScript-first, mas a pergunta interessante
continua a ser arquitectural: **que processo toma a decisão, em que evidência
confia e o que continua a ser aplicável quando o cliente está sob controlo do
utilizador?**

É essa a diferença entre localizar uma verificação e perceber o sistema.

<p>
  <img src="./assets/badboy17jpg.jpg" width="24" height="24" alt="">
  <strong>BadBoy17</strong>
</p>
