# database_benchmark
PostgreSQL + TimeScaleDB: Data compression test for IoT

O teste a seguir foi feito para averiguar a taxa de compressão e velocidade em um banco de dados IoT otimizado para "Time-Series", utilizando PostgreSQL (v12) + TimeScaleDB (2.6.x) com a compressão automática de chunks.

# Tabela
A tabela testada contém as seguintes colunas: TIME (INT), A0 (INT), A1 (INT), ..., A99 (INT).


# Inserindo dados
Foram inseridos dados a fim de simular um dispositivo IoT em campo coletando 25 variáveis a cada 1 segundo durante 1 ano completo. Resumindo, o dispositivo enviou 26 dados ao banco (time + 25) pelo periodo de 01/01/2021 00:00:00 até 01/01/2022 00:00:00, gerando 31536000 linhas com 26 das 101 colunas populadas.

```
INSERT INTO id_123 VALUES (1609470000+generate_series(1, 31536000), random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000, random()*10000);
```

# Benchmark SEM compressão
## Tamanho
```
SELECT HYPERTABLE_SIZE ('id_123');
 hypertable_size 
-----------------
      5971476480 (6GB)
(1 row)
```

## Velocidade
Nota: Foram selecionados 4 meses de dados da coluna A0 (7776000 linhas)

```
EXPLAIN ANALYZE SELECT
  time, A0
FROM id_123
WHERE
  "time" >= 1609470000 AND "time" <= 1617246000
ORDER BY 1



QUERY PLAN                                                                                           
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Custom Scan (ChunkAppend) on id_123  (cost=0.41..14251.94 rows=12991 width=36) (actual time=0.825..6642.660 rows=7776000 loops=1)
   Order: id_123."time"
   ->  Index Scan Backward using _hyper_42_253_chunk_id_123_time_idx on _hyper_42_253_chunk  (cost=0.41..289.52 rows=265 width=8) (actual time=0.818..87.054 rows=161999 loops=1)
         Index Cond: (("time" >= 1609470000) AND ("time" <= 1617246000))
   ->  Index Scan Backward using _hyper_42_254_chunk_id_123_time_idx on _hyper_42_254_chunk  (cost=0.42..4621.66 rows=4242 width=8) (actual time=1.248..1296.839 rows=2592000 loops=1)
         Index Cond: (("time" >= 1609470000) AND ("time" <= 1617246000))
   ->  Index Scan Backward using _hyper_42_255_chunk_id_123_time_idx on _hyper_42_255_chunk  (cost=0.42..4621.66 rows=4242 width=8) (actual time=0.775..1235.954 rows=2592000 loops=1)
         Index Cond: (("time" >= 1609470000) AND ("time" <= 1617246000))
   ->  Index Scan Backward using _hyper_42_256_chunk_id_123_time_idx on _hyper_42_256_chunk  (cost=0.42..4621.66 rows=4242 width=8) (actual time=0.920..1195.836 rows=2430001 loops=1)
         Index Cond: (("time" >= 1609470000) AND ("time" <= 1617246000))
 Planning Time: 1.338 ms
 Execution Time: 6966.372 ms
(12 rows)
```

# Benchmark COM compressão
## Ativando a compressão do TimeScaleDB
Nota: A compressão foi ativa para qualquer chunk mais antigo que 30d.
```
ALTER TABLE id_123 SET (timescaledb.compress);
SELECT set_integer_now_func('id_123', 'unix_now');
SELECT add_compression_policy('id_123', INT '2592000');
```

## Tamanho
```
SELECT HYPERTABLE_SIZE('id_123');
 hypertable_size 
-----------------
      2207727616 (2.2GB)
(1 row)
```

## Velocidade
Nota: Foram selecionados 4 meses de dados da coluna A0 (7776000 linhas)

```
EXPLAIN ANALYZE SELECT
  time, A0
FROM id_123
WHERE
  "time" >= 1609470000 AND "time" <= 1617246000
ORDER BY 1



QUERY PLAN                                                                        
----------------------------------------------------------------------------------------------------------------------------------------------------------
 Custom Scan (ChunkAppend) on id_123  (cost=0.12..137202.60 rows=7777000 width=36) (actual time=378.481..5521.538 rows=7776000 loops=1)
   Order: id_123."time"
   ->  Custom Scan (DecompressChunk) on _hyper_42_253_chunk  (cost=0.12..1639.78 rows=162000 width=8) (actual time=378.470..486.019 rows=161999 loops=1)
         Filter: (("time" >= 1609470000) AND ("time" <= 1617246000))
         ->  Sort  (cost=0.00..0.00 rows=0 width=0) (actual time=377.912..378.803 rows=162 loops=1)
               Sort Key: compress_hyper_69_299_chunk._ts_meta_sequence_num DESC
               Sort Method: quicksort  Memory: 37kB
               ->  Seq Scan on compress_hyper_69_299_chunk  (cost=0.00..13.43 rows=162 width=80) (actual time=377.425..377.774 rows=162 loops=1)
                     Filter: ((_ts_meta_max_1 >= 1609470000) AND (_ts_meta_min_1 <= 1617246000))
   ->  Custom Scan (DecompressChunk) on _hyper_42_254_chunk  (cost=0.14..26285.32 rows=2592000 width=8) (actual time=8.458..574.504 rows=2592000 loops=1)
         Filter: (("time" >= 1609470000) AND ("time" <= 1617246000))
         ->  Sort  (cost=0.00..0.00 rows=0 width=0) (actual time=7.975..10.427 rows=2592 loops=1)
               Sort Key: compress_hyper_69_300_chunk._ts_meta_sequence_num DESC
               Sort Method: quicksort  Memory: 299kB
               ->  Seq Scan on compress_hyper_69_300_chunk  (cost=0.00..211.88 rows=2592 width=80) (actual time=0.055..6.341 rows=2592 loops=1)
                     Filter: ((_ts_meta_max_1 >= 1609470000) AND (_ts_meta_min_1 <= 1617246000))
   ->  Custom Scan (DecompressChunk) on _hyper_42_255_chunk  (cost=0.14..26285.32 rows=2592000 width=8) (actual time=6.580..474.549 rows=2592000 loops=1)
         Filter: (("time" >= 1609470000) AND ("time" <= 1617246000))
         ->  Sort  (cost=0.00..0.00 rows=0 width=0) (actual time=6.498..8.685 rows=2592 loops=1)
               Sort Key: compress_hyper_69_301_chunk._ts_meta_sequence_num DESC
               Sort Method: quicksort  Memory: 299kB
               ->  Seq Scan on compress_hyper_69_301_chunk  (cost=0.00..211.88 rows=2592 width=80) (actual time=0.054..5.323 rows=2592 loops=1)
                     Filter: ((_ts_meta_max_1 >= 1609470000) AND (_ts_meta_min_1 <= 1617246000))
   ->  Custom Scan (DecompressChunk) on _hyper_42_256_chunk  (cost=0.15..24664.67 rows=2431000 width=8) (actual time=3.876..437.739 rows=2430001 loops=1)
         Filter: (("time" >= 1609470000) AND ("time" <= 1617246000))
         Rows Removed by Filter: 999
         ->  Sort  (cost=0.00..0.00 rows=0 width=0) (actual time=3.756..5.438 rows=2431 loops=1)
               Sort Key: compress_hyper_69_302_chunk._ts_meta_sequence_num DESC
               Sort Method: quicksort  Memory: 286kB
               ->  Seq Scan on compress_hyper_69_302_chunk  (cost=0.00..211.88 rows=2431 width=80) (actual time=0.193..2.888 rows=2431 loops=1)
                     Filter: ((_ts_meta_max_1 >= 1609470000) AND (_ts_meta_min_1 <= 1617246000))
                     Rows Removed by Filter: 161
 Planning Time: 4.375 ms
 JIT:
   Functions: 30
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 21.993 ms, Inlining 0.000 ms, Optimization 23.680 ms, Emission 343.764 ms, Total 389.437 ms
 Execution Time: 5997.827 ms
(38 rows)
```

# Considerações finais
A compressão se demonstrou robusta e eficiente, economizando ~63% de memória em disco (-3.8GB), ao mesmo tempo que melhorou a eficiencia ao extrair dados da tabela, ficando ~16% mais rápido (-1seg).
