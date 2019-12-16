# 24 Meses Iniciais

- Infraestrutura de Hardware para IoT

A infraestrutura de IoT é construída sobre sistemas embarcados, tais como sensores, transceptores, e outros dispositivos baseados em microcontroladores de baixo consumo de energia.

Usaremos Arduinos como a principal plataforma de hardware para IoT [4].
A maioria deles é baseada em microcontroladores de baixo consumo da Atmel, tais como o ATmega328p, suportando seis modos standby que podem reduzir o consumo para níveis baixíssimos.
Dependendo das configurações (ex., frequência e voltagem), um Arduino pode drenar de 45mA em operação máxima até 5uA no nível mais profundo de standby.
A literatura mostra que é possível fazer com que as aplicações operem apenas 50% do tempo em média.
Considerando o consumo desprezível de standby, esse comportamento economizaria 50% da bateria.

No ambiente acadêmica, existe bastante pesquisa com o uso de Arduinos no contexto de IoT [4].
A popularidade do Arduino fará com que a nossa pesquisa seja mais acessível e reproduzível para outros grupos.
Em educação, muitos cursos em universidades usam o Arduino [4].
Nós temos usado o Arduino em um curso de graduação pelos últimos 5 anos, o que permitirá avaliar os resultados com programadores de sistemas embarcados menos experientes.
Na comunidade hobista, há uma abundância de software publicamente disponível que poderemos adaptar para a nossa linguagem e avaliar os ganhos de eficiência.

- Infraestrutura de Software para IoT

Em sistemas Arduino (e embarcados em geral), a maneira mais comum de interagir com o mundo externo é através da técnica de "polling", que faz amostragens periódicas em periféricos externos para detectar mudanças de estado.
A técnica de polling gasta ciclos de CPU e previne que o dispositivo entre em modo standby.
No Arduino, mesmo funcionalidades básicas, tais como temporizadores, conversores A/D, e SPI, usam ciclos de polling que gastam energia ininterruptamente em modo ativo.

De modo a prover standby automático, as aplicações deve ser inteiramente reativas a eventos e precisaremos reescrever toda a infraestrutura de software em Céu.
Esse processo consiste primordialmente em reescrever "device drivers", que são os pedaços de software que interagem diretamente com o hardware.
Mais concretamente, no nível mais básico, a eficiência energética depende das rotinas de interrupção (ISRs), que acordam o microcontrolador do modo standby quanto ocorre algum evento em um periférico externo.

Em um projeto anterior, demos os primeiros passos na direção de uma infraestrutura de software em Céu.
Já adicionamos suporte a rotinas de interrupção (ISRs) como um conceito primitivo na nossa linguagem, o que permite reconstruir a infraestrutura com ciência ao modo standby desde sua base.
Também escrevemos os primeiros drivers em Céu para dispositivos que se comunicam com o microcontrolador pelo protocolo SPI.
Essa abordagem não irá afetar a maneira como as aplicações são escritas em níveis mais abstratos, que permanecerão similares às aplicações em Arduino.
No entanto, em vez de gastar ciclos da CPU em espera ativa, as aplicações entrarão no modo de standby mais profundo possível enquanto estiverem ociosas.

O código a seguir é um esboço da abordagem que iremos adotar.
A aplicação solicita, a cada hora, a leitura de um conversor analógico digital e aguarda o seu retorno para executar alguma ação:

```
// application.ceu

output none ADC_REQUEST
input  int  ADC_DONE

#include "adc.ceu" // driver implementation

every 1h do
    emit ADC_REQUEST
    var v = await ADC_DONE
    <executa-alguma-acao>
end
```

É possível notar que o código é escrito de maneira natural, sem callbacks ou qualquer referência ao modo de standby.
As aplicações usam nomes para abstrair os eventos de entrada e saída que são implementados em drivers.
A maior parte do trabalho fica a cargo do driver, que é escrito apenas uma vez e pode ser reusado em todas as aplicações:

```
// adc.ceu

output none ADC_REQUEST do
    <configures-ADC>;
    <enables-ADC-interrupts>;
    <sets-the-deepest-standby-mode>;
end

input int ADC_DONE [ADC_vect_num] do
    <disables-ADC-interrupts>;
    var int v = <reads-adc-data>;
    return v;
end
```

O evento de saída configura o periférico para fazer a requisição, habilita as interrupções correspondentes e informa à linguagem qual é o seu modo de standby mais profundo, mas que ainda permite que o periférico acorde o microcontrolador.
A linguagem é responsável por interagir com os drivers e identificar o maior denominador comum de standby entre todos os dispositivos em uso.
O evento de entrada é acionado pelo dispositivo correspondente através de uma interrupção, acordando a CPU automaticamente.
O código desabilita futuras interrupções do periférico e retorna a leitura requisitada.

Com essa abordagem, o código da aplicação permanece similar aos seus equivalentes em Arduino.
No entanto, em vez de gastar ciclos da CPU com polling, as aplicações irão entrar em standby sempre que estiverem ociosas.
Essa abordagem já foi validada em aplicações e drivers muito simples, mas ainda não há estudos completos por se tratar de projeto em estágio preliminar.

- Aplicações IoT

De modo a avaliar os ganhos de energia com a infraestrutura proposta, precisaremos avaliar o consumo em aplicações realísticas.
A comunidade do Arduino tem uma abundância de projetos open-source que podem ser reescritos na nossa linguagem para tirar proveito do modo de standby transparente.
Então poderemos comparar o consumo de energia entre as versões originais e reescritas para tirar conclusões sobre a efetividade do modo de standby transparente.
Os cenários mais realísticos de IoT usam comunicação por rádio extensivamente.
Nesse contexto, iremos avaliar desde protocolos ad-hoc simples até protocolos mais complexos com ciência energética para ver até que extensão nossa proposta contribuirá efetivamente para a economia de energia.

Os códigos a seguir ilustram o processo de reescrever os programas entre as duas linguagens.
Os dois códigos fazem a mesma coisa: piscam um LED com uma frequência de 1 segundo:

```
// Em C/Arduino
pinMode(13, OUTPUT);
while (1) {
    digitalWrite(13, HIGH);
    delay(1000);
    digitalWrite(13, LOW);
    delay(1000);
}
```

```
// Em Céu
output int PIN_13;
loop do
    emit PIN_13(high);
    await 1s;
    emit PIN_13(low);
    await 1s;
end
```

Note que as chamadas de funções em C, que não carregam nenhuma semântica de eventos, são substituídas por um vocabulário próprio que permite que a linguagem entenda os pontos de espera e possa colocar o microcontrolador em standby.
Em testes preliminares como esse, conseguimos economias de ordem significativa, mas ainda é preciso avaliar aplicações complexas onde há concorrência e uso de múltiplos sensores e atuadores.

No longo prazo, esperamos mostrar para desenvolvedores as vantagens de reescreverem suas aplicações em Céu e tirarem proveito dos modos de standby automaticamente.
Nessa direção, avaliaremos o tempo tomado para reescrever as aplicações e os
ganhos reais de eficiência energética.

# 12 Meses Finais e Trabalhos Futuros

- Arquiteturas e Aplicações de IoT Complexas

O nicho de sistemas embarcados restritos, que inclui o Arduino, cobre a parte substancial (e crescente) de aplicações IoT.
Tipicamente, essas arquiteturas não requerem muitos recursos computacionais e são sensíveis às dimensões físicas e consumo de energia.
No entanto, a IoT também consiste de dispositivos conectados tradicionais, tais como roteadores, servidores e smartphones.
Até 2016 existiam 3.9 bilhões de assinaturas de smartphones no mundo e esse número deve alcançar 6.8 bilhões até 2022 [1].

Smartphones usam arquiteturas muito mais complexas do que microcontroladores embarcados (ex., CPUs de 32-bits com unidade de gerenciamento de memória).
Tipicamente essas arquiteturas dependem de um sistema operacional, uma pilha de TCP/IP completa, e podem executar múltiplas aplicações simultaneamente.
Além de aplicações reativas, típicas de IoT, smartphones também executam computações pesadas, tais como processamento de audio/imagem/vídeo e funções criptográficas.
Mesmo assim, smartphones são uma peça importante na IoT, servindo como uma interface comum aos humanos para processar, visualizar e atuar na rede.

Smartphones têm restrições similares de consumo de bateria e também podem tirar proveito das técnicas propostas para sistemas embarcados restritos.
De modo a transpor a barreira de dispositivos IoT restritos para os smartphones, iremos adotar uma abordagem análoga aos primeiros dois anos:

- Infraestrutura de Hardware:
    Usaremos o BeagleBone Black, que compartilha objetivos similares ao do Arduino, provendo uma plataforma barata e aberta mas que é adequada a aplicações mais ricas, tais como interfaces gráficas, multimídia, e jogos.
- Infraestrutura de Software:
    De modo a garantir standby automático para aplicações, toda infraestrutura de software, principalmente device drivers, também terá que ser recriada usando ISRs em Céu.
- Aplicações:
    Além de aplicações IoT, as aplicações típicas de smartphone, tais como mensagens instantâneas e navegação Web, também podem melhorar a eficiência energética através do modo de standby. Iremos reescrever desde aplicações simples, tais como um relógio gráfico, até aplicações em rede mais complexas, tais como um navegador, para avaliar o consumo de energia.


