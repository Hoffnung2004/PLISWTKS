# 2026-03-31
# Домашнее задание от  2026-03-18
## Введение
В данном домашнем задании был создан цифровой модуль NCO. Была проведена его базовая верфикация. 
## Основная часть
### Используемые коды
Для генерации последовательности и последующей верефикации был создан следующий код на Matlab.
```Matlab
clc
clear
close all
format compact
fid = fopen('output.txt', 'w');
nPA = 7;
nPAN = 6-1;
f_out = 62.5e3;
T = ((1:2^(nPA-2))-0.5)./(2^nPA)*1/f_out;
sin_out = round(sin(2*pi*f_out*T)*(2^nPAN-1))-1;
plot(T,sin_out,'k',LineWidth=2);
grid on
for i = 1:length(T)
fprintf(fid, "%d'd%d: sinus = %d'd%d;\n", [nPA-1 i-1 nPAN+1 sin_out(i)]);
end
fclose(fid);
data = load('sinus_output.txt');
verilog_out = data(1:128)';
verilog_out = verilog_out - mean(verilog_out)+0.01;

figure();
T2 = ((1:2^(nPA))-0.5)./(2^nPA)*1/f_out;
sin_out2 = 2^nPAN+round(sin(2*pi*f_out*T2)*(2^nPAN-1));
sin_out2=sin_out2-mean(sin_out2)+0.01;
plot(sin_out2,'k',LineWidth=2); hold on
plot(verilog_out,'b',LineWidth=2)
subb = verilog_out-sin(2*pi*f_out*T2)*(2^nPAN-1);
subb=subb-mean(subb)+0.01;
plot(subb,'r',LineWidth=2);
xlim([1 128]);
grid on

figure();
plot(10*log10(abs(fftshift(fft(verilog_out))))-33,'k',LineWidth=2); hold on
plot(10*log10(abs(fftshift(fft(sin_out2))))-33,'b',LineWidth=2);
plot(10*log10(abs(fftshift(fft(subb))))-33,'r',LineWidth=2)
grid on
xlim([1 128]);
ylim([-20-33 35-33]);

data = load('sinus_output2.txt');
verilog_out = data(1:128)';
verilog_out = verilog_out-32;

figure();
plot(abs(sin_out2),'k',LineWidth=2); hold on
plot(verilog_out,'b',LineWidth=2);
grid on
xlim([1 128]);
```
Данный код решает несколько задач:
* Генерирует пары фаза - синус для четверти периода синуса
* Сравнивает выход модуля с референсными значениями
* Сравнивает выход модуля с идеальным синусом
* Строит спектры выхода модуля, референсного синуса, разницы между ними. (Спектр идеального синуса не строится)

Данный код является достаточно быстрым наброском, однако имеет особенность, которую следует пояснить. При удалении постоянной составляющей из сигналов, искусственно добавляется новая небольшая составляющая малой величины. Это нужно для того, чтобы при построении спектра не столкнуться с ошибкой при взятии log(0). 

После был написан модуль **PA2PAN** на SystemVerilog.
```Verilog
module PA2PAN(
    input logic [4:0] PA,
    output logic [5:0] sinus
    );
    always_comb begin
        case(PA)
            5'd0: sinus = 6'd0;
            5'd1: sinus = 6'd1;
            5'd2: sinus = 6'd3;
            5'd3: sinus = 6'd4;
			...
            5'd29: sinus = 6'd30;
            5'd30: sinus = 6'd30;
            5'd31: sinus = 6'd30;
        endcase
    end
endmodule
```
Задача данного модуля - конвертация фазы в значение синуса. Следует отметить, что разрядность входной фазы модуля не совпадает с разрядностью аккумулятора фазы. Вызвано это тем, что для записи четверти периода требуется, что очевидно, четверть возможных значений аккумулятора фазы. В данном случае это 32, для передачи которых, требуется 5-битная шина.

Так же для добавления малого шума в выходной сигнал был доработан модуль LFSR из предыдущего домашнего задания.
```Verilog
module LFSR #(
    parameter pol_len = 26 
    )(
    input logic sysclk,
    input logic rst_i,
    output logic out_o
    );
    logic nwe_chislo;
    logic w_clk_out1; 
    logic [0:pol_len-1] pol_i = 26'b10111001011100010000110001;

    logic [0:pol_len-1] registr;
    logic enable;
    assign nwe_chislo = ^ (registr & pol_i);
    always_ff @(posedge sysclk  or posedge rst_i) begin
        if(rst_i) begin
            registr[0:pol_len-1]<=26'b10111001011100010000110001;
        end else begin
            registr<={nwe_chislo,registr[0:pol_len-2]};
        end
    end
    assign out_o=registr[pol_len-1];

endmodule
```
Изменения незначительные. Убраны модули **ILA** и **Clocking Wizard**, работа модуля начинается не с начального состояния. Вызавано это тем, что при текущей реализации, первые 26 бит вывода - нули.

Ниже приведён основной модуль.
```Verilog
module NCO(
    input logic [6:0] increment,
    input logic rst,
    input logic clk,
    output logic [5:0] sinus
    );
    logic lfsr_out;
    logic [6:0] PA;
    logic [4:0] PA_tmp;
    logic [5:0] sinus_tmp;
    always_ff @(posedge rst or posedge clk) begin
        
        if(rst) begin
            PA<=0;
        end else begin
            PA<=PA+increment;
        end
    end
    always_comb begin
        case(PA[6:5])
            2'b00: begin
                PA_tmp = PA[5:0];
                sinus = 6'd33+sinus_tmp^{5'b0,lfsr_out};
            end
            2'b01: begin
                sinus = 6'd33+sinus_tmp^{5'b0,lfsr_out};
                PA_tmp = 5'd31-PA[4:0];
            end
            2'b10: begin
                PA_tmp = PA[5:0];
                sinus = 6'd31-sinus_tmp^{5'b0,lfsr_out};
            end
            2'b11: begin
                PA_tmp = 5'd31-PA[4:0];
                sinus = 6'd31-sinus_tmp^{5'b0,lfsr_out};
            end
        endcase
    end
    LFSR lfsr(
    .sysclk(clk),
    .rst_i(rst),
    .out_o(lfsr_out)
    );
    PA2PAN pa2pan(
    .PA(PA_tmp),
    .sinus(sinus_tmp)
    );
endmodule
```
Данный модуль обеспечивает требуемую по варианту разрядность аккуулятора фазы и выходного сигнала. 
Для вывода синуса целиком используется определённый выше фрагмент синуса. С помощью **case** значения из памяти конвертируются в полный период синуса.

По варианту требуется по мимо синуса сгенерировать сигнал определённой формы.  
РИСУНОК
Для данной задачи можно незначительно модифицировать модуль, изменив анализ текущего значения фазы. Ниже приведён лишь изменённый фрагмент (все изменения были в данном **always_comb**)
Так же был убран модуль LFSR. 
```Verilog
    always_comb begin
        case(PA[6:5])
            2'b00: begin
                PA_tmp = PA[5:0];
                sinus = 6'd33+sinus_tmp;
            end
            2'b11: begin
                PA_tmp = 5'd31-PA[4:0];
                sinus = 6'd33+sinus_tmp;
            end
            2'b01,2'b10: begin
                sinus = 6'd63;
            end
            
        endcase
    end
```
Для возможности верефикации модулей был создан тестбенч, обеспечивающий вывод в файл полученных сигналов.
```Verilog
`timescale 1ns / 1ps

module NCO_tb;

    integer f, f2;

    logic [6:0] increment;
    logic rst;
    logic clk;
    logic [5:0] sinus;
    logic [5:0] sinus2;
    
    NCO dut (
        .increment(increment),
        .rst(rst),
        .clk(clk),
        .sinus(sinus)
    );
     NCO2 dut2 (
        .increment(increment),
        .rst(rst),
        .clk(clk),
        .sinus(sinus2)
    );

    initial clk = 0;
    always #62.5 clk = ~clk;

    initial begin
    
        
        f = $fopen("D:/Standart/Desktop/CONV-BCH/sinus_output.txt", "w");
        if (f == 0) begin
            $display("ERROR: cannot open file");
            $finish;
        end
        f2 = $fopen("D:/Standart/Desktop/CONV-BCH/sinus_output2.txt", "w");
        if (f2 == 0) begin
            $display("ERROR: cannot open file");
            $finish;
        end
        
        rst = 1;
        increment = 0;

        #625;
        rst = 0;

        increment = 7'd1;

        #125000;
        $fclose(f);
        $fclose(f2);
        $finish;
    end
    always @(posedge clk) begin
        if (!rst) begin
            $fwrite(f, "%0d\n", sinus);
            $fwrite(f2, "%0d\n", sinus2);
        end
    end
    initial begin
        $monitor("time=%0t | rst=%0b | inc=%0d | sinus=%0d",
                  $time, rst, increment, sinus);
    end

endmodule
```
По варианту требовалось обеспечить выходную частоту 62.5 кГц. При текущей реализации это можно достичь за счёт подачи на вход NCO тактовой частоты 8 МГц при инкременте равным 1 (что было сделано в тестбенче). В случае, если данная схема будет использоваться для FPGA, данная тактовая частота на входе может быть достигнута за счёт добавления **Clocking Wizard** на схему или добавления своего делителя частоты для использования схемы в составе интегральной микросхемы.

### Вывод кодов
После моделирования и верефикации были получены следующие результаты. 
Временная диаграмма синуса и сигнала по варианту.  
РИСУНОК
Важно отметить, что вернхяя часть сигнала отрисовывается некорректно. На скриншоте она под наклоном, когда как в реальности пологая, что и будет показано на последнем рисунке.
Замер периода сигнала.  
РИСУНОК
Видно, что приод сигнала соответствует требуемой частоте 62.5 кГц.

Референсная форма синуса, заданной в памяти.  
РИСУНОК
Сравнение референсной формы синуса и зашумлённого вывода модуля. (Отдельно было верефицировано, что без зашумления значения идеально совпадают)
Чёрный - референсная форма
Синий - зашумлённый вывод
Красный - разница между идеальным (математическим) синусом. (То есть сумма шума квантования и добавленного шума)  
РИСУНОК
Спектры референсной формы синуса, зашумлённого синуса и шума. (Цвета теже)  
РИСУНОК
Видно, что разница между спектрами синего (зашумлённого) и чёрного (референсного) сигналов незначительно. Отсюда следует, что добавление искуственного шума нецелесообразно при текущей реализации хотя бы при единичном инкременте. И исходя из свойство дискретного преобразования фурье, при инкрементах, являющихся степенью двойки - тоже.
Демонстрация корректности вывода сигнала по варианту.  
РИСУНОК
Для сравнения был показан график модуля синуса.

Так же в лабораторной работе было предложено оценить динамический диапазон, свободный от паразитных составляющих. 
Очевидно, приведённая формула неверна. Так как для 5-битного генератора уже получается 188.72 дБ, что, мягко сказать, много.
При использовании правильной формулы имеем результат $6.02*5-3.92=26.18 дБ$.
## Вывод
В ходе выполнения данного домашнего задания был создан цифровой NCO, который возмонжо использотвать как модуль более сложной системы. 
