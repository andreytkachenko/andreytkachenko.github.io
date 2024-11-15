## Алгоритм быстрого и точного сложения чисел с плавающей точкой

Предположим, у вас есть массив чисел с плавающей запятой, и вы хотите их просуммировать. Вы можете наивно полагать, что можете просто сложить их, например, в Rust:

```rust
fn naive_sum(arr: &[f32]) -> f32 {
    let mut out = 0.0;
    
    for x in arr {
        out += *x;
    }
    
    out
}
```
Однако это может легко привести к сколь угодно большой накопленной ошибке:

```rust
naive_sum(&vec![1.0;     1_000_000]); //  1000000.0
naive_sum(&vec![1.0;    10_000_000]); // 10000000.0
naive_sum(&vec![1.0;   100_000_000]); // 16777216.0
naive_sum(&vec![1.0; 1_000_000_000]); // 16777216.0
```

Проблема в том, что следующим 32-разрядным числом с плавающей запятой после 16777216 является 16777218. Таким образом, когда вы вычисляете 16777216 + 1, оно округляется обратно до ближайшего числа с плавающей запятой с четной мантиссой, которое снова оказывается 16777216. Мы застряли.

### Попарное суммирование

Более разумный метод - использовать попарное суммирование. Вместо полностью линейной суммы с помощью одного накопителя он рекурсивно суммирует массив, разбивая его пополам, суммируя половинки и затем складывая суммы.

```rust
fn pairwise_sum(arr: &[f32]) -> f32 {
    if arr.len() == 0 { return 0.0; }
    if arr.len() == 1 { return arr[0]; }
    let (first, second) = arr.split_at(arr.len() / 2);
    pairwise_sum(first) + pairwise_sum(second)
}
```

Это более точно:

```rust
pairwise_sum(&vec![1.0;     1_000_000]); //    1000000.0
pairwise_sum(&vec![1.0;    10_000_000]); //   10000000.0
pairwise_sum(&vec![1.0;   100_000_000]); //  100000000.0
pairwise_sum(&vec![1.0; 1_000_000_000]); // 1000000000.0
```

Однако это довольно медленно. Чтобы получить процедуру суммирования, которая выполняется как можно быстрее и при этом остается достаточно точной, нам не следует выполнять рекурсию вплоть до массивов длиной 1, так как это приводит к слишком большим затратам на вызовы. Мы все еще можем использовать нашу наивную сумму для небольших размеров и выполнять рекурсию только для больших размеров. Это действительно увеличивает нашу ошибку в худшем случае на постоянный коэффициент, но, в свою очередь, делает попарную сумму почти такой же быстрой, как и наивную сумму.

```rust
fn block_pairwise_sum(arr: &[f32]) -> f32 {
    if arr.len() > 256 {
        let split = (arr.len() / 2).next_multiple_of(256);
        let (first, second) = arr.split_at(split);
        block_pairwise_sum(first) + block_pairwise_sum(second)
    } else {
        naive_sum(arr)
    }
}
```
### Суммирование по Кахану

Ошибка округления в худшем случае при наивном суммировании измеряется с помощью O(nϵ) при суммировании n элементов, где ϵ - машинный эпсилон вашего типа с плавающей запятой (здесь 2^-24). Попарное суммирование улучшает это значение до O((log⁡n)ϵ+nϵ^2). Однако суммирование по Кахану еще больше улучшает это значение до O(nϵ^2), полностью исключая слагаемое ϵ, оставляя только слагаемое ϵ^2, которым можно пренебречь, если только вы не суммируете очень большое количество чисел.

```rust
pub fn kahan_sum(arr: &[f32]) -> f32 {
    let mut sum = 0.0;
    let mut c = 0.0;
    for x in arr {
        let y = *x - c;
        let t = sum + y;
        c = (t - sum) - y;
        sum = t;
    }
    sum
}
```

Суммирование по Кахану работает за счет сохранения суммы в двух регистрах: фактической общей суммы и небольшого значения cc с исправлением ошибок. Если бы вы использовали бесконечно точную арифметику, cc всегда был бы равен нулю, но с плавающей запятой это могло бы быть не так. Недостатком является то, что теперь для добавления каждого числа к сумме требуется четыре операции вместо одной.

Чтобы уменьшить это, мы можем сделать что-то похожее на то, что мы делали с попарным суммированием. Мы можем сначала наивно объединить блоки в суммы, а затем объединить суммы блоков с помощью суммирования по Кахану, чтобы уменьшить накладные расходы за счет точности:

```rust
pub fn block_kahan_sum(arr: &[f32]) -> f32 {
    let mut sum = 0.0;
    let mut c = 0.0;
    for chunk in arr.chunks(256) {
        let x = naive_sum(chunk);
        let y = x - c;
        let t = sum + y;
        c = (t - sum) - y;
        sum = t;
    }
    sum
}
```

### Точное суммирование

Я знаю, по крайней мере, два общих метода для получения правильно округленной суммы последовательности чисел с плавающей запятой. То есть, он логически вычисляет сумму с бесконечной точностью, прежде чем округлить ее обратно до значения с плавающей запятой в конце.
Первый метод основан на примитиве [2Sum](https://en.wikipedia.org/wiki/2Sum), который представляет собой безошибочное преобразование двух чисел x, y в s, t таким образом, что x + y= s + t, где t - небольшая ошибка. Повторяя это до тех пор, пока ошибки не исчезнут, вы сможете получить правильно округленную сумму. Следить за тем, что и в каком порядке нужно добавлять, может быть непросто, и в худшем случае требуется добавить O (n^2), чтобы все термины исчезли. Это то, что реализовано в Python [math.fsum](https://github.com/python/cpython/blob/de19694cfbcaa1c85c3a4b7184a24ff21b1c091(/Modules/mathmodule.c#L1321) и в крейте [fsum](https://docs.rs/fsum/latest/fsum/), которые используют дополнительную память для хранения частичных сумм. Крейт [accurate](https://docs.rs/accurate/latest/accurate/) также реализует это, используя мутацию на месте в [i_fast_sum_in_place](https://docs.rs/accurate/latest/accurate/sum/fn.i_fast_sum_in_place.html).

Другой метод заключается в сохранении большого буфера целых чисел, по одному на экспоненту. Затем, добавляя число с плавающей запятой, вы разлагаете его на экспоненту и мантиссу и добавляете мантиссу к соответствующему целому числу в буфере. Если целое число `buf[i]` переполняется, вы увеличиваете целое число в `buf[i + w]`, где w - ширина вашего целого числа.

На самом деле это позволяет вычислить абсолютно точную сумму без какого-либо округления вообще и фактически является просто чрезмерно допускающим представлением числа с фиксированной запятой, оптимизированного для накопления чисел с плавающей точкой. Этот последний метод занимает минимум времени - O(n), но использует большой, но постоянный объем памяти (≈ 1 КБ для f32, ≈ 16 КБ для f64). Преимущество этого метода в том, что он также является онлайн-алгоритмом - как добавление числа к сумме, так и получение текущей суммы амортизируются O(1).

Вариант этого метода реализован в крейте [accurate](https://docs.rs/accurate/latest/accurate/) как [OnlineExactSum](https://docs.rs/accurate/latest/accurate/sum/struct.OnlineExactSum.html), который использует значения с плавающей запятой вместо целых чисел для буфера.

### Раскрытие возможностей компилятора

Помимо точности, с `naive_sum` есть еще одна проблема. Компилятору Rust не разрешено изменять порядок сложения с плавающей запятой, поскольку сложение с плавающей запятой не является ассоциативным. Таким образом, он не может автоматически векторизовать `naive_sum`, чтобы использовать SIMD-инструкции для вычисления суммы, или использовать параллелизм на уровне команд.

Чтобы решить эту проблему, в Rust есть встроенные функции компилятора, которые вычисляют суммы с плавающей точкой, допуская ассоциативность, такие как [std::intrinsics::fadd_fast](https://doc.rust-lang.org/nightly/std/intrinsics/fn.fadd_fast.html). Однако эти инструкции *невероятно опасны*, поскольку они предполагают, что как входные, так и выходные данные являются конечными числами (никаких бесконечностей, никаких NaN), или в противном случае они представляют собой неопределенное поведение. Это функционально делает их непригодными для использования, поскольку только в самых ограниченных сценариях при вычислении суммы вы знаете, что все входные данные являются конечными числами и что их сумма не может быть переполнена.

Недавно я высказал [Бену Кимоку](https://github.com/saethlin) свое недовольство этими операторами, и мы вместе предложили (и он реализовал) новый набор операторов: [std::intrinsics::fadd_algebraic](https://doc.rust-lang.org/nightly/std/intrinsics/fn.fadd_algebraic.html) и [другие](https://doc.rust-lang.org/nightly/std/intrinsics/fn.fadd_algebraic.html?search=_algebraic).

Я предложил называть операторы алгебраическими, поскольку они допускают (теоретически) любое преобразование, которое оправдано реальной алгеброй. Например, заменив `x − x → 0, cx + cy → c(x + y)` или `x^6 → (x^2)^3`. В общем случае эти операторы обрабатываются как-если они выполняются с использованием вещественных чисел и могут быть сопоставлены любому набору инструкций с плавающей запятой, который был бы эквивалентен исходному выражению, при условии, что инструкции с плавающей запятой были бы точными.

Используя эти новые инструкции, легко сгенерировать автоматически векторизованную сумму:
```rust
#![allow(internal_features)]
#![feature(core_intrinsics)]
use std::intrinsics::fadd_algebraic;

fn naive_sum_autovec(arr: &[f32]) -> f32 {
    let mut out = 0.0;
    for x in arr {
        out = fadd_algebraic(out, *x);
    }
    out
}
```

Если мы скомпилируем с помощью `-C target-cpu=broadwell`, мы увидим, что компилятор автоматически сгенерировал для нас следующий замкнутый цикл, используя 4 аккумулятора и инструкции AVX2:
```asm
.LBB0_5:
    vaddps  ymm0, ymm0, ymmword ptr [rdi + 4*r8]
    vaddps  ymm1, ymm1, ymmword ptr [rdi + 4*r8 + 32]
    vaddps  ymm2, ymm2, ymmword ptr [rdi + 4*r8 + 64]
    vaddps  ymm3, ymm3, ymmword ptr [rdi + 4*r8 + )6]
    add     r8, 32
    cmp     rdx, r8
    jne     .LBB0_5
```

Это позволит обработать 128 байт данных с плавающей запятой (то есть 32 элемента) за 7 команд. Кроме того, все команды vaddps независимы друг от друга, поскольку они накапливаются в разных регистрах. Если мы проанализируем это с помощью uiCA, то увидим, что, по оценкам, указанный цикл занимает 4 цикла, обрабатывая 32 байта за цикл. При частоте 4 ГГц это достигает 128 ГБ/с! Обратите внимание, что это намного превышает пропускную способность оперативной памяти моего компьютера, поэтому вы достигнете такой скорости только при суммировании данных, которые уже находятся в кэше.

Имея это в виду, мы также можем легко определить `block_pairwise_sum_autovec` и `block_kahan_sum_autovec`, заменив их вызовы в `naive_sum` на `naive_sum_autovec`.

### Точность и скорость

Давайте посмотрим, как сравниваются различные методы суммирования. В качестве относительно произвольного критерия давайте просуммируем 100 000 случайных чисел с плавающей точкой в диапазоне от -100 000 до +100 000. Объем данных составляет 400 КБ, поэтому они все еще помещаются в кэш моего AMD Thread ripper 2950x.

Весь код доступен на Github. Скомпилированный с использованием `RUSTFLAGS=-C target-cpu=native` и `--release`, я получаю следующие результаты:

| Алгоритм               | Пропускная способность | Средняя абсолютная погрешность |
|------------------------|------------------------|--------------------------------|
| naive	                 |               5.5 GB/s |                         71.796 |
| pairwise               |               0.9 GB/s	|                         1.5528 |
| kahan                  |               1.4 GB/s |                         0.2229 |
| block_pairwise         |               5.8 GB/s |                         3.8597 |
| block_kahan            |               5.9 GB/s |                         4.2184 |
| naive_autovec          |             118.6 GB/s |                         14.538 |
| block_pairwise_autovec |              71.7 GB/s |                         1.6132 |
| block_kahan_autovec    |              98.0 GB/s |                         1.2306 |
| crate_accurate_buffer  |               1.1 GB/s |                         0.0015 |
| crate_accurate_inplace |               1.9 GB/s |                         0.0015 |
| crate_fsum             |               1.2 GB/s |                         0.0000 |
|----------------------------------------------------------------------------------|


Прежде всего, я хотел бы отметить, что разница в производительности между самым быстрым и самым медленным методом **более чем в 100 раз**. Для суммирования массива! Возможно, это не совсем справедливо, поскольку самые медленные методы вычисляют что-то значительно более сложное, но все равно существует 20-кратная разница в производительности между кажущейся разумной наивной реализацией и самой быстрой.

Мы обнаружили, что в целом методы `_autovac`, использующие `fadd_algebraic`, быстрее и точнее, чем те, которые используют обычное сложение с плавающей запятой. Причина, по которой они также более точны, заключается в той же причине, по которой попарная сумма более точна: любое изменение порядка сложений лучше, поскольку длинная цепочка сложений по умолчанию уже является наихудшим случаем для точности суммы.

Ограничиваясь оптимальным по [Парето](https://ru.wikipedia.org/wiki/%D0%AD%D1%84%D1%84%D0%B5%D0%BA%D1%82%D0%B8%D0%B2%D0%BD%D0%BE%D1%81%D1%82%D1%8C_%D0%BF%D0%BE_%D0%9F%D0%B0%D1%80%D0%B5%D1%82%D0%BE)) выбором, мы получаем следующие четыре реализации:

| Алгоритм               | Пропускная способность | Средняя абсолютная погрешность |
|------------------------|------------------------|--------------------------------|
| naive_autovec          |             118.6 GB/s |                         14.538 |
| block_kahan_autovec    |              98.0 GB/s |                         1.2306 |
| crate_accurate_inplace |               1.9 GB/s |                         0.0015 |
| crate_fsum             |               1.2 GB/s |                         0.0000 |
|----------------------------------------------------------------------------------|

Обратите внимание, что различия в реализации могут быть весьма существенными, и, вероятно, существуют еще десятки методов компенсированного суммирования, которые я здесь не сравнивал.

В большинстве случаев, я думаю, block_kahan_autovec выигрывает здесь, имея хорошую точность (которая не ухудшается при больших входных данных) при почти максимальной скорости. Для большинства приложений дополнительная точность, связанная с правильно округленными суммами, не требуется, и они работают в 50-100 раз медленнее.

Разделив цикл на явный остаток и сжатый цикл суммирования из 256 элементов, мы можем добиться немного большей производительности и избежать пары операций с плавающей запятой для последнего фрагмента:

```rust
#![allow(internal_features)]
#![feature(core_intrinsics)]
use std::intrinsics::fadd_algebraic;

fn sum_block(arr: &[f32]) -> f32 {
    arr.iter().fold(0.0, |x, y| fadd_algebraic(x, *y))
}

pub fn sum_orlp(arr: &[f32]) -> f32 {
    let mut chunks = arr.chunks_exact(256);
    let mut sum = 0.0;
    let mut c = 0.0;
    for chunk in &mut chunks {
        let y = sum_block(chunk) - c;
        let t = sum + y;
        c = (t - sum) - y;
        sum = t;
    }
    sum + (sum_block(chunks.remainder()) - c)
}
```

| Алгоритм               | Пропускная способность | Средняя абсолютная погрешность |
|------------------------|------------------------|--------------------------------|
| sum_orlp               |             112.2 GB/s |                         1.2306 |
|----------------------------------------------------------------------------------|

Вы, конечно, можете изменить число 256, я обнаружил, что использование 128 было на 20% медленнее, а 512 на самом деле не улучшило производительность, но ухудшило точность.

### Заключение

Я думаю, что `fadd_algebraic` и подобные алгебраические встроенные функции очень полезны для создания высокоскоростных подпрограмм с плавающей запятой, и что другие языки также должны добавить их. Параметр `-ffast-math` недостаточно хорош, как мы видели выше, лучшей реализацией был гибрид между автоматически оптимизированной математикой для повышения скорости и реализованными вручную операциями с неассоциативной компенсацией.

Наконец, если вы используете LLVM, остерегайтесь `-ffast-math`. Возможно **неопределенное поведение** - выводить значение `NaN` или `infinity`, пока этот флаг установлен в LLVM. Я понятия не имею, почему они выбрали такую жесткую позицию, которая делает практически каждую программу, использующую его, некорректной. Если вы ориентируетесь на LLVM в своем языке, избегайте [флагов быстрой математики](https://llvm.org/docs/LangRef.html#fastmath) `nan` и `ninf`.

**Оригинальная статья** - [https://orlp.net/blog/taming-float-sums/](https://orlp.net/blog/taming-float-sums/)
