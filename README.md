# Pasos para preprocesar las imagenes en [CPAC](http://fcp-indi.github.io/docs/user/index.html)





1. Generar una carpeta para cada sujeto con la siguiente estrucutura:  "Rs/<sujeto>/ T1/" , Rs/<sujeto>/funcional/"
2.  hacer la corrección de *Slice Timming* `slicetimer` . **Importante:** si la adquisición fue interleaved usar la opcion --odd  
3.  para acceder a cualquier computadora del cluster se hace lo siguiente `ssh -Y usuario@penfield.inb.unam.mx`



** Escribir instrucciones para  Bias filed , copiar la carpeta a ADA , ACCEDER A ADA, CARGAR SINGULARITY, Y CORRER CPAC_GUI**

7. una vez abierto cpac_Gui crear lista de sujetos y pipeline. exportarlo en /mnt/run_cpac/
8. editar archivo .sge el cual debe verse algo así: 

```bash
#! /bin/bash
## SGE batch file - cpacFAB
#$ -S /bin/bash
## cpacFAB is the jobname and can be changed
#$ -N cpac_fab
## execute the job using the mpi_smp parallel enviroment and 8 cores per job
## create an array of 13 jobs the number of subjects
#$ -t 1                                                                                   #ESTO LO DEBES MODIFICAR Y PONER EL NUMERO DE TRABAJOS (1 - "numero de imagenes que se van a procesar")
#$ -V
#$ -l mem_free=2G
#$ -pe openmp 8
#$ -M  jogaru1818@gmail.com                                                                         ### ESCRIBIR AQUI TU DIRECCIÓN DE CORREO ELECTRÓNICO
## change the following working directory to a persistent directory that is
## available on all nodes, this is were messages printed by the app (stdout
## and stderr) will be stored
#$ -wd //mnt/MD1200A/lconcha/afajardo/jonathan/resting_state/run_cpac/working                                       ## ESTO SE DEBE MODIFICAR


### A PARTIR DE AQUI SE ESCRIBEN TODOS LOS COMANDOS PARA PODER CORRER CPAC

module load  singularity/2.2
## sudo chmod 777 /mnt
mkdir -p /mnt/MD1200A/lconcha/afajardo/jonathan/resting_state/run_cpac/log/reports                 

sge_ndx=$(( SGE_TASK_ID - 1 ))

# random sleep so that jobs dont start at _exactly_ the same time
sleep $(( $SGE_TASK_ID % 10 ))

singularity run -B /mnt/MD1200A/lconcha/afajardo/jonathan/resting_state:/mnt -B /mnt/MD1200A/lconcha/afajardo/jonathan/resting_state/tmp:/scratch \
  /mnt/MD1200A/lconcha/afajardo/singularity/singularity_images/bids_cpac-2018-05-30-c1f62374f539.img\
  --n_cpus 2 --mem_gb 8 \
  --pipeline_file /mnt/run_cpac/cpac_pipeline.yml \
  --data_config_file /mnt/run_cpac/cpac_sublist.yml \
  /mnt/data_BIDS \
  /mnt/run_cpac  \
  participant --participant_ndx ${sge_ndx}
[afajardo@naica1 resting_state]$ 

```



9. mandar job a ADA con el comando `qsub cpac_job.sge`
10. para revisar el estado del trabajo se puede con el comando `qstat`



