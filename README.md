# Инструкция по работе с nextflow nf-core пайплайнами

Nextflow — менеджер пайплайнов, позволяющий автоматизировать процессы анализа данных.

nf-core — движимый сообществом проект по унификации биоинформатических пайплайнов, написанных на nextflow.

## Запуск nextflow

### Требования

Nextflow работает на POSIX-совместимых системах (Linux, macOS, и.т.д). На Windows запускается через WSL. Для запуска nextflow требуются bash 3.2 (или более поздней версии), а также java минимум 11 версии.

Можно проверить версию Java:

```bash
java -version
```

Если подходящей версии нет, то её можно установить через SDKMAN!:

1. [Установка SDKMAN](https://sdkman.io/install):

    ```bash
    curl -s https://get.sdkman.io | bash
    ```
    
2. Перезагрузка терминала.

3. Установка Java:

    ```bash
    sdk install java 17.0.10-tem
    ```
### Установка nextflow

1. Nextflow скачивается напрямую:

    ```bash
    curl -s https://get.nextflow.io | bash
    ```

2. Скачавшийся файл делается исполняемым:

    ```bash
    chmod +x nextflow
    ```

3. Программу можно переместить в исполняемый путь `(либо окружение)`:

    ```bash
    sudo mv nextflow /usr/local/bin
    ```

### Обновление

Для обновления используется следующая команда:

```bash
nextflow self-update
```

Для запуска определённой версии (например, 23.10.0) используется следующая команда :

```bash
NXF_VER=23.10.0 nextflow run hello
```

## Откуда брать пайплайны nextflow

При первом запуске nextflow в корневой директории создаётся папка `.nextflow`, куда копируются необходимые для работы программы файлы. Все использованные пайплайны также хранятся в этой папке.

Пайплайны отдельно скачивать `не нужно`. Для запуска пайплайна достаточно указать его репозиторий на github:

```bash
nextflow run nf-core\viralrecon <params>
```

Пайплайны `nf-core` можно посмотреть на их сайте, либо скачать одноимённый пакет _(необязательно)_:

```bash
conda install nf-core
```

И использовать команду

```bash
nf-core list
```

## Модификация пайплайнов

После первого использования в папку .nextflow копируется репозиторий пайплайна. При локальном редактировании файлов, пайплайн не будет запускаться, пока не будет сделан commit.
Но зачастую сам исходный код редактировать необязательно.

Для модификации пайплайнов в nextflow предусмотрены файлы конфигурации:

```bash
nextflow <params> -c custom.config
```

`custom.config` написан синтаксисом nextflow. Например:

```groovy
process {
        withName: '.*:FASTP' {
        memory = 100.GB
    }
        withName: '.*:PANGOLIN' {
        conda = 'bioconda::pangolin=4.2'
    }
        withName: '.*:CONCOCT_CONCOCT' {
        conda = '/home/gorbunov_aa/shared/db/viralrecon_envs/concoct.yaml'
    }

}

conda {
        cacheDir = '/home/gorbunov_aa/shared/db/viralrecon_envs'
}

```

Приведённый выше файл конфигурации для процесса `.*FASTP` ставит параметр операционной памяти на 100 Гб, для `.*PANGOLIN` устанавливает окружение с определённой версией пакета (отличной от указанной в исходном коде), для процесса `.*CONCOCT_CONCOCT` устанавливает окружение в соответсвии с прописанным в файле `concoct.yaml`. Также глобально меняется путь до папки с кэшированными окружениями conda (`cacheDir`). Пайплайны nf-core унифицированы, поэтому файлы конфигурации универсальны.

Ресурсы, используемые процессом, также можно изменять динамически ~~(но надо разбираться в синтаксисе)~~.
[Пример](https://github.com/nf-core/configs/blob/master/conf/crukmi.config):

```groovy
process {
    beforeScript = 'module load apps/apptainer'
    executor     = 'slurm'
    queue        = { task.memory <= 240.GB ? 'compute' : 'hmem' }

    errorStrategy = {task.exitStatus in [143,137,104,134,139,140] ? 'retry' : 'finish'}
    maxErrors     = '-1'
    maxRetries    = 3

    withLabel:process_single {
    cpus   = { check_max( 1 * task.attempt, 'cpus' ) }
    memory = { check_max( 5.GB * task.attempt, 'memory' ) }
    }
}
```

В открытом доступе лежат [файлы конфигураций](https://github.com/nf-core/configs/tree/master/conf) пайплайнов nf-core различных университетов и лабораторий. Можно посмотреть, что делают другие :)

## Окружения

Для запуска каждого этапа пайплайна создаются различные окружения с необходимыми библиотеками. Как эти окружения будут создаваться указывает параметр `profile`:

```bash
nextflow run <params> -profile <conda/docker/singularity/podman/institute>
```

`institute` тут означает один из [файлов конфигурации](https://github.com/nf-core/configs/tree/master/conf) института. 
При выборе профиля docker для каждого этапа подружается и используется соответсвующий образ. 
Профиль conda создаёт в рабочей директории папку с загруженными окружениями. Для каждой рабочей директории окружения создаются заново, поэтому  важно в файле конфигурации указать путь до определённой папки для снижения нагрузки на память сервера, повышения скорости выполнения пайплайнов и контроля версий окружений. Для работы на сервере будет использоваться профиль conda.

## Использование пайплайнов

У каждого пайплайна есть своя страница на сайте [nf-core](https://nf-co.re/pipelines/), где можно посмотреть доступные параметры пайплайнов. 
> Чтобы не запутаться в параметрах: одинарным тире (-) обозначаются параметры nextflow, а двойным (--) параметры nf-core пайплайнов.

Для анализа сиквенсных данных Вектора предполагается использовать 2 пайплайна:
* nf-core/viralrecon для сборки последовательностей на референсный геном
* nf-core/mag для де ново сборки образцов

### viralrecon

viralrecon позволяет собирать и оценивать геномные последовательности из сырых архивированных fastq.gz-файлов. Это буквально формат, в котором данные выгружаются с секвенатора.

Пайплайн оптимизирован разработчиками под вирусы SARS-CoV-2 и MPOX: для уточнения используется параметр `--genome <accession.version>`. Эти аксешники можно найти в [репозитории пайплайна](https://github.com/nf-core/configs/blob/master/conf/pipeline/viralrecon/genomes.config).

Однако, разработчиками также предусмотрена возможность использовать собственные вирусные последовательности. Для этого указывается путь до fasta и gff файлов референсной последовательности: `--fasta /path/to/fasta --gff /path/to/gff`. 

Другие обязательные параметры для запуска пайплайна: `--input`, `--outdir`, `--protocol <metagenomic/amplicon>`, `--platform <illumina/nanopore>`.

При выборе `--protocol amplicon` обязательными становятся параметры `--primer_set` и `--primer_set_version`, которые можно найти в [этом же](https://github.com/nf-core/configs/blob/master/conf/pipeline/viralrecon/genomes.config) репозитории. Также можно указать путь до bed файла с праймерами `--primer_bed </path/to/bed>`

Суммарно, усреднённая команда nextflow выглядит следующим образом:

```terminal
nextflow run nf-core/viralrecon \
    --input samplesheet.csv \
    --outdir <OUTDIR> \
    --platform illumina \
    --protocol amplicon \
    --genome 'MN908947.3' \
    --primer_set artic \
    --primer_set_version 3 \
    --skip_assembly \
    -profile <docker/singularity/podman/conda/institute>
```

### mag

Для mag-пайплайна логика аналогична, усреднённая команда выглядит следующим образом:

```console
nextflow run nf-core/mag \
--input samplesheet.csv \
--outdir <OUTDIR> \
-profile <docker/singularity/podman/shifter/charliecloud/conda/institute>
```

### Формат input'а

Для каждого пайплайна инпутом выступает `csv` таблица с названием и расположением образцов.

`viralrecon`:
```terminal
sample,fastq_1,fastq_2
SAMPLE1_PE,fastq/sample1_R1.fastq.gz,fastq//sample1_R2.fastq.gz
SAMPLE2_PE,fastq/sample2_R1.fastq.gz,fastq/sample2_R2.fastq.gz
SAMPLE3_SE,fastq/sample1_R1.fastq.gz,
SAMPLE3_SE,fastq/sample2_R1.fastq.gz,
```
Скрипт, позволяющий генерировать такие таблицы скачивается отсюда:
```bash
wget -L https://raw.githubusercontent.com/nf-core/viralrecon/master/bin/fastq_dir_to_samplesheet.py
```
Аргументы — папка с расположением fastq.gz файлов _(лучше указывать абсолютный путь)_ и название финальной таблицы:
```bash
./fastq_dir_to_samplesheet.py <FASTQ_DIR> samplesheet.csv
```

`mag`:

Всегда нужна колонка _long reads_, даже если таких прочтений нет.
```terminal
sample,group,short_reads_1,short_reads_2,long_reads
sample1,0,data/sample1_R1.fastq.gz,data/sample1_R2.fastq.gz,
sample2,0,data/sample2_R1.fastq.gz,data/sample2_R2.fastq.gz,
sample3,1,data/sample3_R1.fastq.gz,data/sample3_R2.fastq.gz,
```

Также, если контиги уже были как-то собраны, можно указать путь до этих  fasta.gz файлов:

```terminal
id,group,assembler,fasta
sample1,0,MEGAHIT,MEGAHIT-sample1.contigs.fa.gz
sample1,0,SPAdes,SPAdes-sample1.fasta.gz
sample2,0,MEGAHIT,MEGAHIT-sample2.contigs.fa.gz
sample2,0,SPAdes,SPAdes-sample2.contigs.fasta.gz
sample3,1,MEGAHIT,MEGAHIT-sample3.contigs.fa.gz
sample3,1,SPAdes,SPAdes-sample3.contigs.fasta.gz
```

_Скрипта для генерации `mag` таблиц не завезли, поэтому его надо сделать._

## Команды

Также стоит упомянуть, что команды пайплайнов nextflow можно перечислить в файле с параметрами.

```bash
nextflow run nf-core/viralrecon \
    --input samplesheet.csv \
    --outdir ./results \
    -params-file params.yml \
    -profile docker \
    -r 2.3.1
```

__`params.yml`__

```yml
platform: 'illumina'
protocol: 'amplicon'
primer_set: 'artic'
primer_set_version: 4
genome: 'MN908947.3'
kraken2_db: '/home/harshil/kraken2_human.tar.gz'
kraken2_variants_host_filter: true
min_mapped_reads: 500
save_mpileup: true
variant_caller: 'ivar' 
consensus_caller: 'ivar'
skip_assembly: true
max_memory: '6.GB'
```

### Сборка геномов на референс (полная команда)

```terminal
nextflow run nf-core/viralrecon \
    --input samplesheet.csv \
    --outdir <OUTDIR> \
    --platform illumina \
    --protocol metagenomic \
    --fasta <path/to/fasta> \
    --gff <path/to/gff> \
    --skip_assembly \
    --skip_mosdepth \
    --skip_nextclade \
    --skip_pangolin \
    --skip_asciigenome \
    --skip_kraken2 \
    -profile conda
    -c custom.config
```

Здесь пропущены некоторые этапы пайплайна. Это связано с отсутсвием большой информативности получаемых файлов, с отсутсвием скачанной базы данных или с нерешёнными ошибками, которые на этом шаге появляются.

### Сборка геномов de novo

```terminal
nextflow run nf-core/mag \
    --input samplesheet.csv \
    --outdir <OUTDIR> \
    --run_virus_identification
    --genomad_db <path/to/db> \ 
    --skip_binqc \
    --skip_spades \
    -profile conda \
    -c custom.config
```

Пропуски шагов по тем же причинам.

### Подитог

Таким образом, для запуска пайплайна глобально нужна одна команда. В ней уточняеся конфигурация и параметры, которые могут меняться в зависимости от задачи. Часто требуется доступ к базе данных, которую надо скачать локально. Все референсные последовательности тоже скачиваются локально в силу последних новостей. Есть параметры, которые позволяют подгружать кастомные данные — например, датасет nextclade можно задать через команду `--nextclade_dataset`.

## Кэширование

Каждый запуск nextclade кэшируется. В рабочей директории создаётся папка `work` и `.nextflow`. В первой лежат все логи и промежуточные файлы для каждого этапа пайплайна, во второй — информация по запускам. Каждому запуску присваивается уникальное имя.

Посмотреть все имена:

```bash
nextflow log q
```
Посмотреть какое-то конкретный:
```bash
nextflow log <NAME>
```

> [Подробнее](https://www.nextflow.io/docs/latest/tracing.html)

Благодаря такой системе пайплайн можно запустить с места, где он сломался, или был остановлен. В рабочей директории вводится та же команда, которая была прервана, nextflow сверяет все файлы и продолжает пайплайн с места расхождения файлов с логами. Можно добавлять или убирать какие-то аргументы, менять окружения и пр. — команда будет продолжена с ближайшего доступного места.

```bash
nextflow run <params> -resume
```
> [Подробнее](https://www.nextflow.io/docs/latest/cache-and-resume.html)

Однако, при такой системе временные файлы весят много. Но их можно чистить:

Удалить все временные файлы из всех запусков (логи останутся):
```bash
nexflow clean -k -f
```
Удалить ВСЕ файлы кэша конкретного запуска:
```bash
nexflow clean <NAME> -f
```
> запускается из рабочей директории, [подробнее](https://www.nextflow.io/docs/latest/cli.html#cli-clean).
