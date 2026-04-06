# 2026-04-06
# Домашнее задание от  2026-04-01
## Введение
В данном домашнем задании был создан цифровой КИХ фильтр. Была проведена его базовая верфикация. 
## Основная часть
### Выбор варианта
По требованию 5 варианта следует сделать ФВЧ фильтр с порядком 35, начало полосы пропускания - 10 МГц, частота дескритизации - 180 Мгц.
### Синтез фильтра
Для возможности уложиться в указанный варианте порядок, ограничение для полосы запирания - 32 дБ, а для полосы пропускания 1 дБ. То есть сигнал в полосе запирания затухает хотя бы на 32 дБ, а в полосе пропускания не более чем на 1 дБ.
Ниже приведена АЧХ полученного фильтра.  
![Альтернативный текст](pictures/Screenshot(681).png)  
Ниже приведены, полученные коэфициенты с точностью до 4-го знака после запятой.
```
h = [  
0.0350 0.0067 0.0059 0.0039 0.0007 -0.0037 -0.0095  
-0.0163 -0.0241 -0.0325 -0.0411 -0.0497 -0.0579 -0.0652  
-0.0713 -0.0758 -0.0787 0.9202  
-0.0787 -0.0758 -0.0713 -0.0652 -0.0579 -0.0497 -0.0411  
-0.0325 -0.0241 -0.0163 -0.0095 -0.0037 0.0007 0.0039  
0.0059 0.0067 0.0350  
];
```
### Используемые коды
Ввиду возможной параметризуемости данного фильтра (разный порядок и разрядность) был написан код на C++, который синтезирует модуль фильтра.
```C++
#include<iostream>
#include<string>
#include<fstream>
#include<vector>
#include<cmath>
#define ll long long
#define cout out
using namespace std;


signed main()
{
	int n=20; // 2^-n - error threshold. 2^-20 == 10^-6
	int taps_bit=16;
	double maxA=-1e18;
	ifstream in("input.txt");
	ofstream out("output.txt");
	vector<double>A; // all taps in [-1 1]
	double a;
	while(in>>a) A.push_back(a); 
	
	double tmp=pow(2,n);
	
	// grow to integer
	for(int i=0; i<A.size(); i++)
	{
		A[i]=round(A[i]*tmp);
		maxA=max(maxA,abs(A[i]));
		//cout<<A[i]<<endl;
	}
	
	int fil_bit = ceil(log2(A.size()*maxA))+taps_bit+1;
	
	
	// make head module
	cout<<"module FILTER_FIR("<<endl;
	cout<<"    input  logic clk_i,"<<endl;
	cout<<"    input  logic rst_i,"<<endl;
	cout<<"    input  logic signed ["<<taps_bit-1<<":"<<0<<"] sample_i,"<<endl;
	cout<<"    output logic signed ["<<taps_bit-1<<":"<<0<<"] sample_o,"<<endl;
	cout<<"    output logic valid_o"<<endl;
	cout<<");"<<endl;
	
	
	
	// make reg
	cout<<"logic signed ["<<taps_bit-1<<":"<<0<<"] REGISTER ["<<0<<":"<<A.size()-1<<"];"<<endl;
	
	// make other regs
	cout<<"logic signed ["<<taps_bit-1<<":"<<0<<"] to_MUL1 ["<<0<<":"<<A.size()-1<<"];"<<endl;
	cout<<"logic signed ["<<taps_bit-1<<":"<<0<<"] to_MUL2 ["<<0<<":"<<A.size()-1<<"];"<<endl;	
	cout<<"logic signed ["<<fil_bit-1<<":"<<0<<"] MUL_out ["<<0<<":"<<A.size()-1<<"];"<<endl;
	
	// to summ
	int tmp2 = A.size(), tmp3 = 0;
	while(tmp2>1)
	{
		tmp3++;
		for(int i=0; i<(tmp2+1)/2; i++)
		{
	cout<<"logic signed ["<<fil_bit-1<<":"<<0<<"] summ"<<tmp3<<"_"<<i<<";"<<endl;
		}
		tmp2=(tmp2+1)/2;
	}
	
	
	// make big fucking always_ff
	cout<<"always_ff @(posedge clk_i) begin"<<endl;
	cout<<"    if(rst_i) begin"<<endl;
	cout<<"        valid_o <= 1'b0;"<<endl;
	cout<<"        REGISTER[0] <= {{("<<taps_bit-1<<"){1'b0}}, 1'b1};"<<endl;
	for(int i=1; i<A.size(); i++)
	{
	cout<<"        REGISTER["<<i<<"] <= "<<taps_bit<<"'b0;"<<endl;	
	}
	for(int i=0; i<A.size(); i++)
	{
	cout<<"        to_MUL1["<<i<<"] <= "<<taps_bit<<"'b0;"<<endl;	
	}
	for(int i=0; i<A.size(); i++)
	{
	cout<<"        to_MUL2["<<i<<"] <= "<<taps_bit<<"'b0;"<<endl;	
	}
	for(int i=0; i<A.size(); i++)
	{
	cout<<"        MUL_out["<<i<<"] <= "<<fil_bit<<"'b0;"<<endl;	
	}
	tmp2 = A.size();
	tmp3 = 0;
	while(tmp2>1)
	{
		tmp3++;
		for(int i=0; i<(tmp2+1)/2; i++)
		{
	cout<<"        summ"<<tmp3<<"_"<<i<<"<="<<fil_bit<<"'b0;"<<endl;
		}
		tmp2=(tmp2+1)/2;
	}
	cout<<"    end"<<endl;
	cout<<"    else begin"<<endl;
		// shift register
	cout<<"        REGISTER[0] <= sample_i;"<<endl;
	for(int i=1; i<A.size(); i++)
	{
	cout<<"        REGISTER["<<i<<"] <= REGISTER["<<i-1<<"];"<<endl;
	}
	cout<<endl;
	
	// pipeline before multiplication
	for(int i=0; i<A.size(); i++)
	{
	cout<<"        to_MUL1["<<i<<"] <= REGISTER["<<i<<"];"<<endl;
	}
	cout<<endl;
	
	for(int i=0; i<A.size(); i++)
	{
	cout<<"        to_MUL2["<<i<<"] <= to_MUL1["<<i<<"];"<<endl;
	}
	cout<<endl;
	
	// multiplication by coefficients
	for(int i=0; i<A.size(); i++)
	{
	cout<<"        MUL_out["<<i<<"] <= to_MUL2["<<i<<"] * ";
	if(A[i]<0) cout<<"-";
	cout<<(n+2)<<"'sd"<<abs((ll)A[i])<<";"<<endl;
	}
	cout<<endl;
	
	// pipelined adder tree
	int prev_cnt = A.size();
	int stage = 1;
	int curr_cnt;
	
	while(prev_cnt > 1)
	{
		curr_cnt = (prev_cnt + 1) / 2;
		
		for(int i=0; i<prev_cnt/2; i++)
		{
			if(stage==1)
			{
	cout<<"        summ"<<stage<<"_"<<i<<" <= MUL_out["<<(2*i)<<"] + MUL_out["<<(2*i+1)<<"];"<<endl;
			}
			else
			{
	cout<<"        summ"<<stage<<"_"<<i<<" <= summ"<<stage-1<<"_"<<(2*i)<<" + summ"<<stage-1<<"_"<<(2*i+1)<<";"<<endl;
			}
		}
		
		if(prev_cnt % 2)
		{
			if(stage==1)
			{
	cout<<"        summ"<<stage<<"_"<<(curr_cnt-1)<<" <= MUL_out["<<(prev_cnt-1)<<"];"<<endl;
			}
			else
			{
	cout<<"        summ"<<stage<<"_"<<(curr_cnt-1)<<" <= summ"<<stage-1<<"_"<<(prev_cnt-1)<<";"<<endl;
			}
		}
		
		cout<<endl;
		prev_cnt = curr_cnt;
		stage++;
	}
	
	// output
	cout<<"        sample_o <= summ"<<stage-1<<"_0["<<taps_bit-1+n<<":"<<n<<"];"<<endl;
	cout<<"        valid_o <= valid_o | (|(summ"<<stage-2<<"_0 + summ"<<stage-2<<"_1));"<<endl;
	
	cout<<"    end"<<endl;
	cout<<"end"<<endl;
	cout<<endl;
	cout<<"endmodule"<<endl;
}   
```
Идея проста. Подсчёт требуемой разрядности, чтобы гарантировать отсутствие переполнения и возможной потери точности на округлениях.
Фактически это очень ограничивает параметры в срипте. Больше порядок фильтра - больше рахрядность, а тут уже и так значение, близкое к пределу.

Для 5-го варианта был синтезирован следующий 16-битный фильтр.
```Verilog
module FILTER_FIR(
    input  logic clk_i,
    input  logic rst_i,
    input  logic signed [15:0] sample_i,
    output logic signed [15:0] sample_o,
    output logic valid_o
);
logic signed [15:0] REGISTER [0:34];
logic signed [15:0] to_MUL1 [0:34];
logic signed [15:0] to_MUL2 [0:34];
logic signed [42:0] MUL_out [0:34];
logic signed [42:0] summ1_0;
...
logic signed [42:0] summ6_0;
always_ff @(posedge clk_i) begin
    if(rst_i) begin
        valid_o <= 1'b0;
        REGISTER[0] <= {{(15){1'b0}}, 1'b1};
        REGISTER[1] <= 16'b0;
		...
        REGISTER[33] <= 16'b0;
        REGISTER[34] <= 16'b0;
        to_MUL1[0] <= 16'b0;
        ...
        to_MUL1[34] <= 16'b0;
        to_MUL2[0] <= 16'b0;
        ...
        to_MUL2[34] <= 16'b0;
        MUL_out[0] <= 43'b0;
        ...
        MUL_out[34] <= 43'b0;
        summ1_0<=43'b0;
        ...
        summ6_0<=43'b0;
    end
    else begin
        REGISTER[0] <= sample_i;
        REGISTER[1] <= REGISTER[0];
        ...
        REGISTER[34] <= REGISTER[33];

        to_MUL1[0] <= REGISTER[0];
        ...
        to_MUL1[34] <= REGISTER[34];

        to_MUL2[0] <= to_MUL1[0];
        ...
        to_MUL2[34] <= to_MUL1[34];

        MUL_out[0] <= to_MUL2[0] * 22'sd36746;
        ...
        MUL_out[34] <= to_MUL2[34] * 22'sd36746;

        summ1_0 <= MUL_out[0] + MUL_out[1];
        ...
        summ1_16 <= MUL_out[32] + MUL_out[33];
        summ1_17 <= MUL_out[34];

        summ2_0 <= summ1_0 + summ1_1;
        ...
        summ2_8 <= summ1_16 + summ1_17;

        summ3_0 <= summ2_0 + summ2_1;
        ...
        summ3_4 <= summ2_8;

        summ4_0 <= summ3_0 + summ3_1;
        summ4_1 <= summ3_2 + summ3_3;
        summ4_2 <= summ3_4;

        summ5_0 <= summ4_0 + summ4_1;
        summ5_1 <= summ4_2;

        summ6_0 <= summ5_0 + summ5_1;

        sample_o <= summ6_0[35:20];
        valid_o <= valid_o | (|(summ5_0 + summ5_1));
    end
end

endmodule

```
Код представлен в сокращённом виде. Общая идея такая.
Так как тактовая частота - 180 МГц, все суммы и умножения нужно считать быстро. Поэтому был построен небольшой конвеер. По задумке умножения должны выполнять DSP ядра, что быстрее, чем на лутах.
После синтеза видно, что DSP ядра были действительно задействованы.  
![Альтернативный текст](pictures/Screenshot(682).png)  
Это и является слабым местом реализации. Так как ядра имеют ограниченный размер операндов умножения, существенно увеличить битность кода без потери скорости не представляется возможным без модифицирования кода.

Так же выход фильтра имеет сигнал валидности. Для того, чтобы его отслежить без постройки дополнительного регистра, по сигналу reset по фильтру пускается единичка. Таким образом первое ненулевое число (до обрезки лишних бит) на выходе - сигнал, что пошли валидные данные. При данной реализации пущенная единичка, является предельно малой величиной и на выходе фильтра просто не ощущается.

Для верификации кода был создан данный testbench.
```Verilog
`timescale 1ns/1ps

module tb_FILTER_FIR;

    // ---------------- параметры ----------------
    parameter CLK_FREQ  = 180_000_000;
    parameter SIN_FREQ  = 703125;
    parameter AMP       = 10000;

    localparam real CLK_PERIOD_NS = 1e9 / CLK_FREQ;

    // ---------------- сигналы ----------------
    logic clk_i;
    logic rst_i;
    logic [15:0] max_o;
    logic [15:0] min_o;
    logic [15:0] diff_o;
    logic [31:0] delay_cnt;
    logic delay_done;

    // signed (реальный сигнал)
    logic signed [15:0] sample_i_signed;
    logic signed [15:0] sample_o_signed;

    // offset версия (для отображения)
    logic [15:0] sample_i;
    logic [15:0] sample_o;

    logic valid_o;

    // ---------------- DUT ----------------
    FILTER_FIR dut (
        .clk_i(clk_i),
        .rst_i(rst_i),
        .sample_i(sample_i_signed),
        .sample_o(sample_o_signed),
        .valid_o(valid_o)
    );

    // ---------------- clock ----------------
    initial begin
        clk_i = 0;
        forever #(CLK_PERIOD_NS/2.0) clk_i = ~clk_i;
    end

    // ---------------- синус ----------------
    real t;
    real dt;

    initial begin
        dt = 1.0 / CLK_FREQ;
        t  = 0.0;
    end

    function automatic signed [15:0] gen_sin;
        real val;
        begin
            val = AMP * $sin(2.0 * 3.1415926535 * SIN_FREQ * t);
            gen_sin = $rtoi(val);
        end
    endfunction

    // ---------------- DC offset ----------------
    // добавляем смещение: +2^15
    always_comb begin
        sample_i = sample_i_signed + 16'sd32768;
        sample_o = sample_o_signed + 16'sd32768;
    end

    // ---------------- стимулы ----------------
    initial begin
        rst_i = 1;
        sample_i_signed = 0;

        repeat(10) @(posedge clk_i);
        rst_i = 0;

        forever begin
            @(posedge clk_i);

            sample_i_signed <= gen_sin();
            t = t + dt;
        end
    end
    
    always_ff @(posedge clk_i) begin
    if (rst_i) begin
        delay_cnt  <= 0;
        delay_done <= 1'b0;
    end
    else begin
        if (!delay_done) begin
            delay_cnt <= delay_cnt + 1;

            if (delay_cnt >= 18000)
                delay_done <= 1'b1;
        end
    end
end
    
assign diff_o = max_o - min_o;

always_ff @(posedge clk_i) begin
    if (rst_i) begin
        max_o <= 16'd0;
        min_o <= 16'hFFFF;
    end
    else if (delay_done) begin
        if (sample_o > max_o)
            max_o <= sample_o;

        if (sample_o < min_o)
            min_o <= sample_o;
    end
end
    
    
endmodule
```
Данный тестбенч позволяет направить на вход фильтра сигналы заданной частоты и амплитуды. Так же измеряется размах выходного сигнала, что позволяет построить АЧХ полученного фильтра.
### Выводы кодов
После моделирования фильтра были полученны различные временные диаграммы. 
Пример такой диаграммы приведён ниже.  
![Альтернативный текст](pictures/Screenshot(683).png)  
Видно, что сигнал низкой частоты (703125 Гц) подавляется.

По полученным результатам временных моделирований была получена следующие точки на АЧХ.

| № | Размах (peak-to-peak) | Амплитуда | Коэффициент пропускания, дБ |
|---|------------------------|-----------|-----------------------------|
| 1 | 20137                  | 10068.5   | 0.06                        |
| 2 | 21121                  | 10560.5   | 0.47                        |
| 3 | 20783                  | 10391.5   | 0.34                        |
| 4 | 4629                   | 2314.5    | -12.71                      |
| 5 | 29                     | 14.5      | -56.77                      |
| 6 | 495                    | 247.5     | -32.13                      |
| 7 | 543                    | 271.5     | -31.32                      |

Ниже приведено сравнение с теоретической АЧХ.  
![Альтернативный текст](pictures/Screenshot(684).png)  

Видно, что АЧХ созданного фильтра в точности совпадает (в точках замера) с теоретической АЧХ из Matlab. 
Возможно, для полной верфикации такого числа точек недостаточно. Но даже сейчас видно, что провал в АЧХ есть и у синтезированного фильтра, высокие частоты проходят без искажений, а нулевые частоты ослабеваются на 30 дБ, что соответствует начальной задумке.
## Вывод
В ходе выполнения домашнего задания... всё работает штатно. 
