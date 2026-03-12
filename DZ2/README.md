# 2026-03-12
# Домашнее задание от  2026-02-18
## Введение
В данном домашнем задании было выполнено создание модуля РЛОС, для генерации m-последовательности, считывание с помощью ILA сгенерированной модулем последовательности и сравнение её с результарами, сформированными с помощью референсной модели языке Python.
## Основная часть
### Выбор варианта
Ниже представлен python-код, который генерирует полином в соответствии с номером варианта. 
 ```python
import pylfsr as pyl

var = 5

print(list(list(pyl.get_fpolyList(m=26) + pyl.get_fpolyList(m=27) + pyl.get_fpolyList(m=28) + pyl.get_fpolyList(m=29)))[var])
 ```
Варианту 5 соответствует следующий полином: 26, 23, 22, 21, 19, 18, 15, 14, 13, 11, 10, 9, 8, 6, 5, 2.
### Написание модуля на SystemVerilog
Ниже приведено описание модуля на SystemVerilog. Данный модуль не имеет выводов, так как модуль ILA подключён внутри модуля. Для получения информации из модуля можно провод **out_o** объявить выходным.
Параметр **pol_i** так же можно сделать входным проводом, обеспечив таким образом, полностью параметризируемый модуль.
 
```verilog
`timescale 1ns / 1ps

module LFSR #(
    parameter pol_len = 26
    )(
    input logic sysclk,
    input logic rst_i
    );
    logic out_o;
    logic nwe_chislo;
    logic w_clk_out1;
    logic [0:pol_len-1] pol_i = 26'b01001_10111_10111_00110_11100_1;

    logic [0:pol_len-1] registr;
    logic enable;
    assign nwe_chislo = ^ (registr & pol_i);
    always_ff @(posedge w_clk_out1) begin
        if(rst_i) begin
            registr[0]<=1'b1;
            registr[1:pol_len-1]<=0;
        end else begin
            registr<={nwe_chislo,registr[0:pol_len-2]};
        end
    end
    assign out_o=registr[pol_len-1];
    ila_0 u_ila0 (
        .clk    (w_clk_out1),
        .probe0 (out_o),
        .probe1 (registr),
        .probe2 (rst_i)
    );
    clk_wiz_0 clkk(
        .clk_in1(sysclk),
        .clk_out1(w_clk_out1)
    );

endmodule
```
### Работа с ILA
Для полного понимания процессов внутри модуля ILA считывает не только **out_o**, но и значения регистра и сигнал сброса. 
Для отслеживания именно начала последовательности триггер был установлен на **rst_i == 0**. 
Таким образом после отжатия кнопки сброса, ILA считывает первые 4096 бит последовательности.
![Альтернативный текст](pictures/Screenshot(667).png)
Информация, полученная от ILA, была экспортирована в csv-файл. 
![Альтернативный текст](pictures/Screenshot(670).png)
Дальше требуемый столбец был скопирован в файл **out.txt**.
### Сравнение с референсной моделью
Для сравнения с референсной моделью был использован следующий python-код.
```python
import pylfsr as pyl
import numpy as np

file_path = 'D:/Standart/Desktop/DZ/out.txt'

with open(file_path, 'r') as file:

    simulation_result = file.read() # Содержимое файла результата моделирования

simulation_result=simulation_result[0::2]

file.close()

state = [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0] # Начальное состояние

fpoly = [26, 23, 22, 21, 19, 18, 15, 14, 13, 11, 10, 9, 8, 6, 5, 2] # Коэффициенты полинома

lfsr_4 = pyl.LFSR(fpoly=fpoly,initstate=state, verbose=False, counter_start_zero=False)

lfsr_4.runKCycle(20000)

simulation_result_list = np.array([int(i[0]) for i in simulation_result])

gold_res = lfsr_4.get_outputSeq()

errors = abs(simulation_result_list - gold_res[0:len(simulation_result_list)]);

print(sum(errors) ) # Сравнение сигналов моделей на Verilog и Python
```
Его вывод был 0. Таким образом есть все основания полагать, что модуль РЛОС написан корректно. Так как если не было ошибки на первых 4096 итерациях, то их не будет и дальше. Действительно. В общем случае данная логика не верна, хотя иногда и может применяться. Однако для РЛОС длинной 26 или менее, данное умозаключение имеет место быть. 
## Вывод
В ходе выполнения данной лабораторной работы был реализован РЛОС для генерации m-последовательности. Данный модуль представляет из себя базовый функционал, так как выдаёт 1 бит за такт, что во многих случаях может быть недостаточно. Однако на его примере получилось разобрать 2 этапа работы над проектом. Создание и верефикация, воможно исправление в случае, если верефикация не была пройдена.
