# Гайд по парсингу таблицы и генерации файлов на вход пайплайна

## Универсальная команда

```bash
nextflow run nf-core/pipeline --input_sheet —-outdir -profile conda -c custom.config -params-file params.yml
```

* nf-core/pipeline — имя репозитория на гитхабе
>можно использовать локальный репозиторий или гитлаб при дополнительной настройке
input_sheet — таблица со всеми файлами, которые будут обрабатываться
* outdir — папка, куда будет сохраняться результат пайплайна 
> на уровень выше этой папки всегда лежат папки с кэшем work и .nextflow, а также логи .nextflow.log
*profile указывает, что надо использовать конду
* c — файл конфигурации
* params-file — файл параметров пайплайна

__`Достаточно генерировать файл с конфигурацией и параметрами`__

## Это делается по общей БД (таблице)

Таблица делится на 3 логических части:
* files
* conditions
* config

Во всех таблицах есть колонки для обозначения:
* состояний интерфеса (interface в названии)
* их связь с параметрами пайлайна (parameter; parameter_arg)
* зависимости параметров (если какой-то параметр не может быть запущен без другого)

### files

здесь перечислены все доступные для вируса селекторы

то есть варианты референса, праймеров, хостов и nexclade’а (type_interface) для каждого вируса (virus_interface)
для дальнейшего использования вынесены отдельные бинарные колонки для каждого вируса

interface condition показывает в каком состоянии может находится элемент интерфейса — в этой таблице все **select

### conditions

все доступные для вируса бинарные аргументы

указаны все доступные состояния (true/false) с соответствующими параметрами пайплайна, их связь с интерфейсом, зависимости

по большей части как прошлая таблица, но благодаря interface_condition можно получить параметр пайплайна в зависимости от выбора пользователя

также заполнена колонка default_interface с дефолтным значением параметра в интерфейсе

__ВАЖНО__

interface_parameter и parameter_arg часто _противоположны_!!
отсюда доступ к параметрам пайплайна из связи с параметрами интерфейса,
а не параметры интерфейса напрямую

### config

таблица с параметрами конфигурации
по ней можно генерировать мокап файла конфигурации с помощью create_config

это самая неочевидная таблица, потому что parameters и parameters_arg здесь не для параметров интерфейса, а для параметров команд в конфигурации

этот файл пишется на DSL языке джавы, который используется nextflow

поэтому тут есть level, step, config_arg колонки

### пример конфигурации

#### 1. conda
`этот кусок с conda указывает путь, куда сохранять и где искать окружения`
```bash
conda {
	cacheDir = '/mnt/dbs_nextflow/nextflow_resources/nf-core/envs/viralrecon_env/'
}
```

в таблице у этого куска нет parameter, но есть 
parameter_arg =  '/mnt/dbs_nextflow/nextflow_resources/nf-core/envs/viralrecon_env/'
level = conda
config_step пустой
config_arg = cacheDir

2. #### process
`этот кусок указывает на процессы, здесь берётся FASTP`
```bash
process {
	withName: 'FASTP' {
		conda = 'bioconda::fastp=0.23.4'
		ext.args = '--length_required 30 --length_limit 0 --cut_mean_quality 20 --trim_poly_x  --cut_front --cut_tail'
	}
}
```

level = process

config_step = FASTP — шаг пайплайна

config_arg = conda

parameter = «»

config_arg = ext.args

 parameter = length_required

  parameter_arg = 30

 parameter = length_limit

  parameter_arg = 0

_итд_

## КАК ГЕНЕРИРУЕТСЯ ИНТЕРФЕЙС

### СЕЛЕКТОРЫ

- прописываются вручную
- на вход все доступные для вируса селекторы из отфильтрованной по вирусу таблице
- можно для аргументов парсинга прописать соответствующий parameter_interace

> если вариант выбора 1, его можно дефолтным выводить

### КОЛИЧЕСТВЕННЫЕ

- текстовое поле, можно прописать в цикле
- аргументы парсинга — parameter_interface

> в таблице есть колонка default_interface, где прописаны дефолтные значения

### БИНАРНЫЕ

- чекбоксы, можно прописать в цикле
- аргументы парсинга — parameter_interface
- дефолтные значения из default_interface

### «ЗАБАНЕННЫЕ»

- не выводятся в интерфейс, аргументы парсинга всегда False

### КАК ГЕНЕРИРУЕТСЯ ФАЙЛ С ПАРАМЕТРАМИ

	изначальная таблица — table
	cортировка таблицы по ВИРУСУ — filtered_table

#### СЕЛЕКТОРЫ (Ref, Trim Primers, Remove Host Read, Nextclade)

	table —> filtered_table —> index by type_interface (indexed_f_table)
	indexed_f_table AT [type_interface = user_arg] —> parameter: parameter_arg
_user_arg — ввод пользователя_
> селекторы уникальны по типам, поэтому это работает

! если селектора не представлено, то почти всегда это требует бинарного аргумента в другом параметре интерфейса, поэтому в отсутствие селектора/выбора из селектора

interface_condition = False

БИНАРНЫЕ ПАРАМЕТРЫ

	filtered_table —> filter only binary with isinstance(interface_condition, bool) —> index by interface_condition —> select AT [interface_condition = user_arg] —> parameter: parameter_arg
- тут нужно указать False (interface_condition) для всех «забанненых» параметров

- у аргумента Virus Prediction странные параметры, поэтому для него нужно сделать исключение и отфильтровать исходя из зависимостей

#### ВЫБОР ВИРУСА

можно аргументы парсинга назвать fasta и gff и просто написать
> value — выбранный файл

	user_arg: value


#### КОЛИЧЕСТВЕННЫЕ ПАРАМЕТРЫ (либо по остаточному принципу)

почти как селекторы, только вместо type interface берётся interface_parameter
> полученные ключ-значение parameter: parameter_arg сохраняются в yml-формате

	1. filtered_table —> index by interface_parameter (indexed_f_table) 

	2. indexed_f_table AT [interface_parameter = user_arg] —> parameter: user_value


##### пример:

```
gff: /mnt/dbs_nextflow/references/viruses/orthomyxoviridae/influenza_a/h3n2_2024/GCA_040230115.1_ASM4023011v1_genomic.240611.gff.gz
fasta: /mnt/dbs_nextflow/references/viruses/orthomyxoviridae/influenza_a/h3n2_2024/GCA_040230115.1_ASM4023011v1_genomic.240611.fna.gz
bowtie2_index: /mnt/dbs_nextflow/references/viruses/orthomyxoviridae/influenza_a/h3n2_2024/GCA_040230115.1_ASM4023011v1_genomic.240611.fna.index.gz
kraken2_db: /mnt/dbs_nextflow/databases/kraken2/homo_sapiens.GRCh38.p14.220203.k2.tar.gz
primer_bed: /mnt/dbs_nextflow/references/primers/influenza/influenza_a.primers.artic.h3n2.bed
nextclade_dataset: /mnt/dbs_nextflow/databases/nextclade/orthomyxoviridae/influenza_a_h3n2/ha/240808.zip
min_mapped_reads: 100
skip_fastqc: false
skip_variants_quast: false
skip_picard_metrics: false
skip_snpeff: false
ivar_trim_noprimer: false
save_unaligned: false
skip_multiqc: true
skip_variants_long_table: true
skip_pangolin: true
```

### КАК ГЕНЕРИРУЕТСЯ КОНФИГ

- есть заготовки, которые сгенерированы скриптом create_config.py

- в эти заготовки подставляется соответствующий menu_id из таблицы параметр со значением

#### БИНАРНЫЕ

	1. table —> table_filtered by interface_parameter —> table_f_indexed by interface_parameter

	2. table_f_indexed AT [interface_parameter = user_arg] —> parameter: id_menu
- тут берётся именно parameter, а не parameter_arg, потому что это флаг bash-команды

- parameter форматируется в --parameter

#### КОЛИЧЕСТЕННЫЕ (сейчас это все остальные)

	1. filtered_table —> index by interface_parameter (indexed_f_table) 

	2. indexed_f_table AT [interface_parameter = user_arg] —> id_menu: user_value


- с помощью string template в заготовку подставляются полученные аргументы конфига по ключу id_menu
