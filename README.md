# Simulador de treino Aqualibrium

Simulador da divisão de 3 L de água entre três reservatórios por uma rede de tubos montada sobre uma malha de 4 × 4 pontos. Este documento descreve o modelo físico e os métodos numéricos empregados.

## 1. Representação da rede

A malha é tratada como um grafo com 16 nós e 24 arestas de comprimento $L$. Cada aresta pode estar vazia ou ocupada por um tubo de diâmetro interno $D_6$ ou $D_3$. Além dos nós da malha, o modelo inclui duas condições de contorno de carga conhecida:

- a fonte, com carga $H_s(t)$, ligada ao ponto 1 pelo tubo principal;
- a atmosfera na entrada de cada reservatório, com carga $-z_{out}$, ligada ao nó correspondente por uma saída vertical.

A rede é considerada horizontal. Assim, as cargas de posição dos nós são iguais e a diferença de carga entre dois nós corresponde apenas à diferença de pressão.

## 2. Conservação de massa

Em cada nó $i$ da rede, a soma das vazões que saem é igual à soma das que entram:

$$
F_i(\mathbf{H}) = \sum_{j \in \mathcal{V}(i)} Q_{ij}(H_i - H_j) + Q_{out,i} - Q_{in,i} = 0
$$

em que $\mathcal{V}(i)$ é o conjunto de nós ligados a $i$ por tubos, $Q_{out,i}$ é a vazão para o reservatório (quando existe) e $Q_{in,i}$ é a vazão vinda da fonte (apenas no ponto 1).

## 3. Perda de carga em cada trecho

A relação entre a diferença de carga $h$ e a vazão $Q$ em um trecho combina perda distribuída e perda localizada:

$$
h = f \frac{L}{D} \frac{V^2}{2g} + K \frac{V_c^2}{2g}, \qquad V = \frac{Q}{A}, \qquad V_c = \frac{Q}{A_c}
$$

O primeiro termo é a equação de Darcy-Weisbach, que representa o atrito ao longo do tubo. O segundo representa as perdas nos conectores. Como o diâmetro interno do conector ($D_c$) é menor que o do tubo de 6 mm, a contração e a expansão na passagem pelo conector dominam a perda localizada, e por isso $K$ é referido à velocidade $V_c$ dentro do conector.

O fator de atrito é obtido por:

$$
f = \max\left(\frac{64}{Re},\ 0{,}316\,Re^{-1/4}\right), \qquad Re = \frac{V D}{\nu}
$$

O primeiro termo corresponde ao escoamento laminar (Hagen-Poiseuille) e o segundo à correlação de Blasius para tubos lisos em regime turbulento. O uso do máximo produz uma transição contínua entre os regimes, em $Re \approx 1.190$, o que mantém a função $h(Q)$ contínua e crescente e garante a convergência do método numérico.

Para o cálculo da rede é necessária a relação inversa, $Q(h)$. No ramo laminar, a perda distribuída é linear em $Q$ e a equação resultante é do segundo grau, com solução explícita:

$$
h = aQ + bQ^2 \ \Rightarrow\ Q = \frac{-a + \sqrt{a^2 + 4bh}}{2b}, \qquad a = \frac{32\,\nu L}{g D^2 A}, \qquad b = \frac{K}{2 g A_c^2}
$$

No ramo turbulento, $h = cQ^{7/4} + bQ^2$, e $Q$ é obtido por iterações de Newton.

## 4. Solução da rede: método de Newton-Raphson

As cargas dos nós são as incógnitas do sistema não linear $\mathbf{F}(\mathbf{H}) = \mathbf{0}$. A cada iteração resolve-se o sistema linear:

$$
\mathbf{J}\,\Delta\mathbf{H} = -\mathbf{F}(\mathbf{H}), \qquad J_{ii} = \sum_{j} \frac{\partial Q_{ij}}{\partial h}, \qquad J_{ij} = -\frac{\partial Q_{ij}}{\partial h}
$$

A matriz jacobiana tem a mesma estrutura de uma matriz de rigidez: cada tubo contribui com sua condutância incremental $\partial Q / \partial h$ na diagonal dos dois nós e com o valor oposto fora dela. As ligações com a fonte e com os reservatórios contribuem apenas na diagonal. O sistema é resolvido por eliminação de Gauss com pivoteamento parcial, e o passo $\Delta\mathbf{H}$ é limitado para evitar oscilações. O processo termina quando o maior resíduo de vazão fica abaixo de $10^{-11}$ m³/s.

## 5. Esvaziamento da fonte

A carga da fonte diminui à medida que ela esvazia:

$$
H_s(t) = z_t + \frac{V(t)}{A_s}, \qquad \frac{dV}{dt} = -\sum_{k=1}^{3} Q_k
$$

em que $z_t$ é a cota da base da fonte em relação à malha e $A_s$ é a área da seção da fonte. Adota-se a hipótese de regime quase permanente: como a rede responde muito mais rápido do que a fonte esvazia, em cada instante a rede é resolvida como se a carga $H_s$ fosse constante.

A integração no tempo é feita por passos explícitos, com $\Delta t$ escolhido para que cada passo transfira no máximo 20 mL. O último passo é ajustado para que a simulação termine exatamente quando o volume total recebido pelos reservatórios atinge 3 L.

## 6. Penalidade

O desempenho da montagem é medido pela soma dos desvios absolutos em relação ao volume ideal de cada reservatório:

$$
P = \sum_{k=1}^{3} \left| V_k - 1000\ \text{mL} \right|
$$

## 7. Verificação das regras do kit

O tipo de conector em cada nó é definido pelo número de bocas, igual ao número de tubos ligados ao nó acrescido de uma boca vertical no ponto 1 e nos pontos dos reservatórios: duas bocas exigem um reto (tubos alinhados) ou um cotovelo, três exigem um tê e quatro, uma cruzeta. Nós com uma única boca representam pontas abertas, e nós com cinco bocas são inviáveis. O simulador verifica ainda os limites de tubos de cada diâmetro, o número máximo de arestas vazias, o limite de conectores de cada tipo e a ligação de todos os reservatórios ao ponto 1.

## 8. Busca da melhor montagem

A busca é feita por recozimento simulado (*simulated annealing*). A partir de uma montagem aleatória, cada iteração propõe uma alteração, que pode ser a troca do estado de uma aresta ou a permuta entre duas arestas. A nova montagem é avaliada pela função de custo:

$$
C = 3000 \sum_{k=1}^{3} \left| s_k - \tfrac{1}{3} \right| + 300\,E + \lambda\,(n - 8)
$$

em que $s_k$ é a fração da vazão destinada ao reservatório $k$, calculada em regime permanente com a fonte a meio volume, $E$ é o número de violações das regras do kit e $n$ é o número de trechos retos uniformes da montagem. O termo $\lambda(n - 8)$ favorece redes em que linhas e colunas inteiras têm o mesmo tipo de tubo, mais fáceis de memorizar e montar; o mínimo possível é $n = 8$.

Alterações que reduzem o custo são sempre aceitas. As que aumentam o custo são aceitas com probabilidade dada pelo critério de Metropolis:

$$
p = \exp\left(-\frac{\Delta C}{T}\right), \qquad T_i = T_0 \left(\frac{T_f}{T_0}\right)^{i/N}
$$

A temperatura decresce de $T_0 = 300$ a $T_f = 1$ ao longo de $N = 2000$ iterações, com quatro reinícios independentes. A aceitação de pioras temporárias permite escapar de mínimos locais.

As dez melhores montagens encontradas são então simuladas de forma completa, com esvaziamento da fonte, em três cenários de perda localizada: os coeficientes $K$ nominais, reduzidos à metade e aumentados em 50%. É escolhida a montagem que minimiza a penalidade no pior cenário somada a $\lambda(n - 8)$, o que privilegia soluções robustas à incerteza da calibração. Os estilos "Mais precisa", "Equilibrada" e "Mais simples" correspondem a $\lambda$ igual a 0, 12 e 30 mL, respectivamente.

## 9. Parâmetros padrão

| Parâmetro | Símbolo | Valor |
|---|---|---|
| Comprimento de cada trecho | $L$ | 0,30 m |
| Diâmetro interno do tubo grande | $D_6$ | 6 mm |
| Diâmetro interno do tubo pequeno | $D_3$ | 3 mm |
| Diâmetro interno dos conectores | $D_c$ | 3 mm |
| Cota da base da fonte acima da malha | $z_t$ | 0,40 m |
| Lâmina d’água na fonte com 4 L | $h_s$ | 0,10 m |
| Desnível da malha à entrada dos reservatórios | $z_{out}$ | 0,15 m |
| Coeficiente de perda, trecho de 6 mm | $K_6$ | 1,0 |
| Coeficiente de perda, trecho de 3 mm | $K_3$ | 0,4 |
| Coeficiente de perda na saída | $K_{out}$ | 1,5 |
| Viscosidade cinemática da água | $\nu$ | 0,89 × 10⁻⁶ m²/s |

Todos os valores podem ser alterados na seção **Calibração do modelo**.

## 10. Hipóteses e limitações

- Rede horizontal, com tubos rígidos e água a temperatura constante.
- Regime quase permanente, sem efeitos transitórios nem presença de ar na rede.
- Volume de água retido nos tubos desprezado.
- Perda localizada uniforme por trecho, sem distinção entre os tipos de conector.
- Coeficientes $K$ estimados. Antes da calibração com medições no kit real, o modelo representa bem as tendências de distribuição, mas os volumes absolutos devem ser interpretados como aproximações.
