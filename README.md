

# Preprocesamiento "Standard" en [CPAC](https://fcp-indi.github.io/docs/user/index.html)

# Resting State

### **preprocesamiento de las imágenes funcionales**:

1. Corrección de *Slice Timming*
2. Corrección de movimiento
3. *skull strip*
4. Cálculo de Framewise displacement (método de Power)
5. **regresión del ruido:**
- regresión de la señal sustancia blanca (valor promedio por volumen)
- regresión de la señal del líquido cerebroespinal ( valor promedio por volumen)
- Compcor(primeros 6 componentes) [Behzadi et al., 2007](https://www.sciencedirect.com/science/article/pii/S1053811907003837?via%3Dihub)
- linear & cuadratic detrending (controlar efecto de algunos artefactos inducidos por el funcionameinto del resonador y el movimiento del sujeto)
- parámetros de movimiento (Friston 24)
- Despiking: Si un volumen tiene un valor de framewise displacement mayor a **0.25**  es incluído dentro del modelo de regresión. Todos los sujetos incluídos en el analísis deben contar con al menos `n` volúmenes.
6. Filtrado temporal: Filtro pasabandas para las frecuencias de 0.01 - 0.08 Hz [Satherthwaite et al., 2013](https://www.sciencedirect.com/science/article/pii/S1053811912008609?via%3Dihub)
7. Registro linear con imagén estructural (T1w) y transformación no lineal a espacion MMNI.
8. Suavizado espacial: FWHM de 6mm


# <u>Pasos para correr [CPAC](http://fcp-indi.github.io/docs/user/index.html)</u>



1.  Generar una carpeta de trabajo (de preferencia con las imágenes en formato BIDS) con la siguiente estructura:
```
   resting_state/

   ├── data_BIDS
   │   └── sub-001  <----- (Una carpeta para cada sujeto)
   │       ├── anat
   │       │   └── sub-001_T1.nii.gz
   │       └── func
   │           └── sub-001_task-rest_bold.nii.gz
   ── run_cpac
   │   ├── crash
   │   ├── log
   │   ├── output
   │   └── working
   └── tmp
   ```
2. Hacer la corrección de *Slice Timming*  de la imagen funcional con fsl 5.0.11

   ```bash
   slicetimer -i sub-001_task-rest_bold.nii.gz -o sub-001_task-rest_bold.nii_stc.gz --odd
   ```

   **NOTA: la opción** `--odd`  s**e emplea en imagenes con adquisición Interleaved.**     

   3.  Si es necesario, hacer *bias field correcction* de la T1 de la siguente manera con FSL:

      - Estimar el bias field:

      ```bash
      fast -b -o sub-001_T1w_bias.nii.gz sub-001_T1w.nii.gz
      ```

   - Dividir la imagen estructural entre la imagen de bias_field:

     ```bash
     fslmaths ub-001_T1w.nii.gz -div sub-001_T1w_bias.nii.gz sub-001_T1w_bcorr.nii.gz
     ```

4.  Copiar la carpeta de trabajo al directorio de trabajo en lavis:

   ```bash
   scp -rv resting_state afajardo@ada.lavis.unam.mx:/mnt/MD1200A/lconcha/afajardo/jonathan
   ```

   *(teclear la contraseña para acceder a lavis)*

5. Acceder a través de una red de la UNAM a lavis y cambiarse al directorio de la carpeta de trabajo.

   ``` bash
   ssh -Y afajardo@ada.lavis.unam.mx
   ```

   ```bash
   cd /mnt/MD1200A/lconcha/afajardo/jonathan
   ```

   6. Crear el pipeline y la lista de sujetos a traves de CPAC GUI. para esto se invoca el contenedor de singularity; se monta la carpeta de trabajo dentro del contenedor y se incluyen como argumentos posicionales la carpeta de inputs y la carpeta donde se desean generar los outpus. Los comandos empleados deben ser algo similar a esto:

      ```bash
      module load singularity/2.2
      ```

``` bash
singularity run -B /mnt/MD1200A/lconcha/afajardo/jonathan/resting_state/:/mnt -B /mnt/MD1200A/lconcha/afajardo/jonathan/resting_state/tmp:/scratch /mnt/MD1200A/lconcha/afajardo/singularity/singularity_images/bids_cpac-2018-05-30-c1f62374f539.img --skip_bids_validator /mnt/data_BIDS /mnt/run_cpac GUI

```

- Para crear lista de sujetos se da click en `New` y se  especifican los directorios en donde se encuentran las imagenes. En este caso como no estamos usando el formato BIDS la pantalla debe verse como en la imagen. Si se introdujeron los argumentos sin errores al hacer click en el boton  `Generate Data Config` la ventana debe cerrarse automaticamente y visualizaremos una pestaña la tista de sujetos (hacer click en `View`para inspeccionarla)

  ![](/home/alfonso/Documentos/github/fMRI/GUI.png)

- Generar un nuevo pipeline dando click en `New`. Revisar la documentación de CPAC para modificar los parametros deseados. guardarlo en /mnt/run_cpac.

- cerrar CPAC GUI y revisar  el pipeline y la lista. para esto hacer `cd`al directorio donde se generon los archivos y visualizarlos con `gedit cpac_pipeline.yml &`. **Importante verificar que esta linea sea igual a esta:** 

  ```python
  # (Motion Spike De-Noising only) Number of volumes to de-spike or scrub preceding a volume with excessive FD.
  numRemovePrecedingFrames :  0
  
  
  # (Motion Spike De-Noising only) Number of volumes to de-spike or scrub subsequent to a volume with excessive FD.
  numRemoveSubsequentFrames :  0
  
  ```

7. Editar archivo cpac_jog .sge el cual debe verse algo así (*verificar que los nombres de la lista de sujetos y del pipeline sean los correctos* ): 

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
```



9. Mandar job a ADA con el comando: 

   - `qsub cpac_job.sge`

   - para revisar el estado del trabajo se puede con el comando `qstat`. Debe verse en el estado del trabajo `r`

10. Una vez se hayan procesados las imagenes copiar la carpeta a tu computadora con el siguiente comando

    ```bash
    scp -rv afajardo@ada.lavis.unam.mx:/mnt/MD1200A/lconcha/afajardo/jonathan/resting_state/run_cpac/output .
    ```

    *Las imagnes ya preprocesadas pueden encontrarse en las subcarpetas functional_to_standard y/o functional_freq_filtered. Tambien pueden revisarse las carpetas Despiking_frames_ para ver  los volúmenes que se excluyeron.*

    

    11. Realizar el suavizado espacial  de cada imagen a través de los siguientes pasos:

    - Crear una máscara del cerebro

      **Con FSL:**

```bash
bet bandpassed_demeaned_filtered_warp.nii.gz   sub-001  -m 
```

**Con AFNI (recomendado):** 

 ```bash
3dAutomask -prefix sub-001_mask.nii.gz bandpassed_demeaned_filtered_warp.nii.gz
 ```



Suavizar sobre la máscara. Con **FSL** se usa `fslmaths`con la opcion `-s` 

**Con AFNI:** 

```bash
3dBlurInMask -input bandpassed_demeaned_filtered_warp.nii.gz -prefix bandpassed_demeaned_filtered_warp_FWHM6.nii.gz -mask sub-001_mask.nii.gz -FWHM 6
```

12: Hacer control de calidad de las imagenes y revisar si se cumple con los criterios de inclusion.
