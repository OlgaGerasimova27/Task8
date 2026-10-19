# Task8
Рубежный практический контроль: Комплексный практикум по базовым конструкциям C#
### Вариант 15. Управление роботизированным складом зерна

1. **Побитовый блок:** Упаковать в `int` номер силосной башни (5 бит, 1–32), процент заполнения (7 бит, 0–100%), температуру зерна (7 бит, 0–60 C), влажность (7 бит, 0–50%) и флаг риска самосогревания (1 бит).
2. **Матрица:** В матрице температуры насыпи зерна `int[5, 5]` найти максимальный элемент и переместить его в центр матрицы `[2, 2]`, поменяв местами с центральным элементом.
3. **Метод (ref, out, params):** Метод `VentilateSilos(ref int[] temps, ref int[] humidities, out int dangerSilosCount, params int[] tempLimits)`: включает принудительное охлаждение башен, снижает температуру на 3 градуса во всех проблемных силосах и возвращает их количество.
4. **Рекурсия:** Рекурсивный расчет времени разгрузки зерна самотеком через воронку `CalculateUnloadTime(int currentVolumeTons, int dischargeRatePerHour)`.
5. **Перегрузка:** Семейство методов `CalculateGrainValue`:
   - `CalculateGrainValue(int tons, int pricePerTon)` — базовая стоимость партии;
   - `CalculateGrainValue(int tons, int pricePerTon, int moisturePercent, int trashPercent, out int discountRubles)` — расчет с дисконтом за повышенную влажность и сорность зерна.

---
```csharp
using System;

class Program
{
    // ======================= MAIN =======================
    static void Main()
    {
        Console.WriteLine("=== ЭТАП 1: Побитовая упаковка ===");
        Stage1_Packing();

        Console.WriteLine("\n=== ЭТАП 2: Трансформация матрицы ===");
        Stage2_Matrix();

        Console.WriteLine("\n=== ЭТАП 3: VentilateSilos (ref/out/params) ===");
        Stage3_Ventilate();

        Console.WriteLine("\n=== ЭТАП 4: Рекурсия разгрузки ===");
        Stage4_Recursion();

        Console.WriteLine("\n=== ЭТАП 5: Перегрузка CalculateGrainValue ===");
        Stage5_Overload();

        Console.ReadKey();
    }

    // ============================================================
    // ЭТАП 1. Упаковка: tower(5) | fill(7) | temp(7) | hum(7) | risk(1)
    // Разряды:  31..27    26..20    19..13    12..6     5
    // ============================================================
    static void Stage1_Packing()
    {
        int tower = 17;      // 1..32 (5 бит)
        int fill  = 82;      // 0..100 (7 бит)
        int temp  = 34;      // 0..60  (7 бит)
        int hum   = 27;      // 0..50  (7 бит)
        int risk  = 1;       // 0..1   (1 бит)

        int packed = PackSilo(tower, fill, temp, hum, risk);
        Console.WriteLine("Упаковано: 0x" + ToHex(packed) + " (" + packed + ")");

        int t2, f2, tm2, h2, r2;
        UnpackSilo(packed, out t2, out f2, out tm2, out h2, out r2);
        Console.WriteLine("Распаковано: tower=" + t2 + " fill=" + f2 +
                          " temp=" + tm2 + " hum=" + h2 + " risk=" + r2);

        // Контрольный бит чётности: считаем единичные биты без деления
        int ones = CountBits(packed);
        Console.WriteLine("Единичных бит: " + ones + " -> чётность: " + (ones & 1));
        Console.WriteLine("Контроль чётности (ones&1==0): " + ((ones & 1) == 0));
    }

    static int PackSilo(int tower, int fill, int temp, int hum, int risk)
    {
        int packed = 0;
        packed |= (tower & 0x1F);        // 5 бит
        packed |= (fill  & 0x7F) << 5;   // 7 бит
        packed |= (temp  & 0x7F) << 12;  // 7 бит
        packed |= (hum   & 0x7F) << 19;  // 7 бит
        packed |= (risk  & 0x01) << 26;  // 1 бит
        return packed;
    }

    static void UnpackSilo(int packed, out int tower, out int fill,
                           out int temp, out int hum, out int risk)
    {
        tower = packed & 0x1F;
        fill  = (packed >> 5)  & 0x7F;
        temp  = (packed >> 12) & 0x7F;
        hum   = (packed >> 19) & 0x7F;
        risk  = (packed >> 26) & 0x01;
    }

    static int CountBits(int v)
    {
        int count = 0;
        int x = v;
        for (int i = 0; i < 32; i++)
        {
            count += (x & 1);
            x = x >> 1;
        }
        return count;
    }

    static string ToHex(int v)
    {
        char[] hex = { '0','1','2','3','4','5','6','7','8','9','A','B','C','D','E','F' };
        char[] buf = new char[8];
        int x = v;
        for (int i = 7; i >= 0; i--)
        {
            buf[i] = hex[x & 0xF];
            x = (int)((uint)x >> 4);
        }
        return new string(buf);
    }

    // ============================================================
    // ЭТАП 2. Найти максимум в int[5,5] и поменять с центром [2,2]
    // ============================================================
    static void Stage2_Matrix()
    {
        int rows = 5, cols = 5;
        int[,] m = new int[rows, cols];
        // Заполним детерминированно (без Random для воспроизводимости)
        int seed = 7;
        for (int i = 0; i < rows; i++)
        {
            for (int j = 0; j < cols; j++)
            {
                seed = (seed * 1103515245 + 12345) & 0x7FFFFFFF;
                m[i, j] = seed % 90 + 5; // 5..94
            }
        }
        Console.WriteLine("Исходная матрица:");
        PrintMatrix(m, rows, cols);

        int maxI = 0, maxJ = 0;
        int maxV = m[0, 0];
        for (int i = 0; i < rows; i++)
        {
            for (int j = 0; j < cols; j++)
            {
                if (m[i, j] > maxV) { maxV = m[i, j]; maxI = i; maxJ = j; }
            }
        }
        Console.WriteLine("Максимум " + maxV + " в (" + maxI + "," + maxJ + ")");

        int ci = rows / 2, cj = cols / 2; // [2,2]
        int buf = m[ci, cj];
        m[ci, cj] = m[maxI, maxJ];
        m[maxI, maxJ] = buf;

        Console.WriteLine("После перемещения максимума в центр:");
        PrintMatrix(m, rows, cols);
    }

    static void PrintMatrix(int[,] m, int rows, int cols)
    {
        for (int i = 0; i < rows; i++)
        {
            for (int j = 0; j < cols; j++)
            {
                Console.Write(m[i, j].ToString().PadLeft(4));
            }
            Console.WriteLine();
        }
    }

    // ============================================================
    // ЭТАП 3. VentilateSilos(ref, out, params)
    // Проблемный силос: temp > любой из лимитов (проверяем по max лимита)
    // Снижаем температуру на 3, считаем количество проблемных.
    // ============================================================
    static void Stage3_Ventilate()
    {
        int[] temps      = { 22, 41, 18, 55, 33, 47, 29 };
        int[] humidities = { 40, 60, 35, 70, 50, 65, 45 };

        Console.WriteLine("До вентиляции:");
        PrintArr("temps     ", temps);
        PrintArr("humidity  ", humidities);

        int danger;
        VentilateSilos(ref temps, ref humidities, out danger, 35, 45, 50);

        Console.WriteLine("Опасных силосов: " + danger);
        Console.WriteLine("После вентиляции:");
        PrintArr("temps     ", temps);
        PrintArr("humidity  ", humidities);
    }

    static void VentilateSilos(ref int[] temps, ref int[] humidities,
                               out int dangerSilosCount, params int[] tempLimits)
    {
        dangerSilosCount = 0;
        int n = temps.Length;
        if (humidities.Length < n) n = humidities.Length;

        // Максимальный порог из params (если параметров нет — берём 40)
        int limit = 40;
        if (tempLimits != null && tempLimits.Length > 0)
        {
            limit = tempLimits[0];
            for (int i = 1; i < tempLimits.Length; i++)
            {
                if (tempLimits[i] > limit) limit = tempLimits[i];
            }
        }

        for (int i = 0; i < n; i++)
        {
            if (temps[i] > limit)
            {
                temps[i] = temps[i] - 3;
                // Принудительное охлаждение также снижает влажность на 2%
                humidities[i] = humidities[i] - 2;
                if (humidities[i] < 0) humidities[i] = 0;
                dangerSilosCount++;
            }
        }
    }

    static void PrintArr(string name, int[] a)
    {
        Console.Write(name + ": ");
        for (int i = 0; i < a.Length; i++) Console.Write(a[i] + " ");
        Console.WriteLine();
    }

    // ============================================================
    // ЭТАП 4. Рекурсия: CalculateUnloadTime(currentVolumeTons, dischargeRatePerHour)
    // Время разгрузки самотеком. Если объём <= 0 -> 0.
    // Каждый час выгружается rate, но с падением эффективности (воронка):
    // фактическая скорость = rate - (rate/10)*hourIndex (не ниже 1).
    // Возвращаем суммарное время в часах (целое, округление вверх).
    // ============================================================
    static void Stage4_Recursion()
    {
        int vol = 250;
        int rate = 40;
        int hours = CalculateUnloadTime(vol, rate, 0);
        Console.WriteLine("Объём=" + vol + " т, скорость=" + rate + " т/ч -> время=" + hours + " ч");
    }

    static int CalculateUnloadTime(int currentVolumeTons, int dischargeRatePerHour)
    {
        return CalculateUnloadTime(currentVolumeTons, dischargeRatePerHour, 0);
    }

    static int CalculateUnloadTime(int currentVolumeTons, int dischargeRatePerHour, int hour)
    {
        if (currentVolumeTons <= 0) return 0;
        if (dischargeRatePerHour <= 0) return 0;

        // Эффективность падает на 10% базовой скорости за каждый час
        int effective = dischargeRatePerHour - (dischargeRatePerHour / 10) * hour;
        if (effective < 1) effective = 1;

        int removed = currentVolumeTons < effective ? currentVolumeTons : effective;
        int remain = currentVolumeTons - removed;

        if (remain <= 0) return hour + 1;
        return CalculateUnloadTime(remain, dischargeRatePerHour, hour + 1);
    }

    // ============================================================
    // ЭТАП 5. Перегрузка CalculateGrainValue
    // ============================================================
    static void Stage5_Overload()
    {
        int tons = 120;
        int price = 15000;

        int baseValue = CalculateGrainValue(tons, price);
        Console.WriteLine("Базовая стоимость: " + baseValue + " руб.");

        int discount;
        int net = CalculateGrainValue(tons, price, 18, 4, out discount);
        Console.WriteLine("С дисконтом: " + net + " руб. (скидка " + discount + " руб.)");

        // Дополнительная перегрузка: стоимость по массиву тонн и массиву цен
        int[] tonsArr  = { 50, 80, 120, 200 };
        int[] priceArr = { 15000, 14500, 14000, 13800 };
        int total = CalculateGrainValue(tonsArr, priceArr);
        Console.WriteLine("Суммарная стоимость по партиям: " + total + " руб.");
    }

    // Базовая
    static int CalculateGrainValue(int tons, int pricePerTon)
    {
        if (tons <= 0 || pricePerTon <= 0) return 0;
        return tons * pricePerTon;
    }

    // С дисконтом за влажность (>14%) и сорность (>2%)
    static int CalculateGrainValue(int tons, int pricePerTon,
                                   int moisturePercent, int trashPercent,
                                   out int discountRubles)
    {
        discountRubles = 0;
        if (tons <= 0 || pricePerTon <= 0) return 0;

        int baseCost = tons * pricePerTon;
        int moisturePenalty = 0;
        int trashPenalty = 0;

        if (moisturePercent > 14)
        {
            // 1% скидки за каждый % влажности сверх 14
            moisturePenalty = (moisturePercent - 14) * baseCost / 100;
        }
        if (trashPercent > 2)
        {
            // 2% скидки за каждый % сорности сверх 2
            trashPenalty = (trashPercent - 2) * baseCost / 50; // = *2/100
        }

        discountRubles = moisturePenalty + trashPenalty;
        int result = baseCost - discountRubles;
        return result < 0 ? 0 : result;
    }

    // Перегрузка: суммарная стоимость по массивам (одинаковой длины)
    static int CalculateGrainValue(int[] tons, int[] prices)
    {
        if (tons == null || prices == null) return 0;
        int n = tons.Length;
        if (prices.Length < n) n = prices.Length;

        int total = 0;
        for (int i = 0; i < n; i++)
        {
            int part = CalculateGrainValue(tons[i], prices[i]);
            total += part;
        }
        return total;
    }
}
```
