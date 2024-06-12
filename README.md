# Deploy OAI-RAN

Nesse repositório são tratados os diferentes tipos de implementação da RAN provida pela OAI (Open Air Interface) que é composta pela gNodeB e UE.

## Tipos de implementação
Temos diferentes tipos de implementação da RAN provida pela OAI, desse modo, elas estão separadas nas seguintes documentações:

### Bare-metal:
* É possível compilar e implementar a gNodeB em conjunto com as UEs em uma máquina utilizando seus recursos, o passo a passo dessa implementação pode ser feita usando a seguinte [documentação](serviceModels/Docs/Imp-bare-metal.md).


## Docker:
* Também é possível utilizar containers docker para realizar a implementação da RAN com diversas UEs, esse deploy permite uma maior portabilidade para diferentes ambientes e maior escala para testes com diversas UEs. O passo a passo dessa implementação se encontra no seguinte [tutorial](serviceModels/Docs/Imp-docker.md)