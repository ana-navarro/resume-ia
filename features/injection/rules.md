# Injeção de Documentos para Alimentar a IA

Regras de sistemas de injeção de documentos

## Gerais
- Só deve aceitar documentos .pdf
- Não possuíra um front responsável então deve ter um endpoint público para isso no backend
- Todos os nomes dos atributos tem que ser em inglês

### Apresentação
A ideia dos PDFs de apresentação é alimentar a IA com assuntos fora dos currículos. E também ter uma aba no menu principal de apresentação no frontend que vai carregar este arquivo para o usuário mostrar sua apresentação e apresentar ela para o recruiter.

#### Regras
- Só é permitido uma apresentação por língua
- Endpoint separado para subida de upload apresentação
    - Deve receber este endpoint o arquivo, idioma da apresentação e substituicao
    - No banco de dados, deve salvar o nome do arquivo, data de upload, se é o mais atual e linguagem.
    - Toda vez que o usuário sobe um novo, se tiver um com a mesma língua, deve enviar o atributo substituicao (atributo opcional) true no endpoint para permitir a substituição, se não enviar o atributo ou colocar como falsedeve retornar um erro.
    - Quando acontece a substituição deve criar um objeto de auditoria.
- Endpoint para pegar esse documento filtrando por língua.
- Endpoints de auditoria separados.
- O documento aqui enviado deve ficar armazenado na pasta assets e dentro dela deve possui duas pastas uma para currículos e outra para apresentações. Deve ser armazenados em apresentação.

### Currículos
A ideia aqui é armazenar todos os currículos do mais antigo para o mais novo. Ele será responsável por alimentar a IA para qualidades técnica da pessoa. Além de, também possuir uma aba para mostrar o currículo atual para o recruiter. Como vai ter vários .PDFs, alguns podem ter mais informações que os outros

##### Regras
- Pode ter vários documentos de currículos
- Endpoint separado para a subida de upload currículo
    - Deve receber este endpoint o arquivo, idioma, confirmacao e tipo (pode ser contexto ou atualizado)
    - A versão mais atual ou com o tipo atual, só pode existir uma
    - Quando sobe um com o tipo atual, deve enviar um atributo (confirmacao, atributo opcional) para confirmação. Se o usuário confirmar todos os outros viram contexto. Se o usuário não confirmar ou não enviar nada, deve retornar um erro.
- Endpoint para pegar esse documento filtrando por língua.
- Endpoints de auditoria separados.
- O documento aqui enviado deve ficar armazenado na pasta assets e dentro dela deve possui duas pastas uma para currículos e outra para apresentações. Deve ser armazenado em currículos.
    - O unico ponto, somente o ultimo curriculo com atualizado deve ser armazenado na pasta. O resto para a IA descobri com embeddings.
