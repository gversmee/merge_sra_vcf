
## How to merge vcf files from TOPMed studies
Step 1: After creating a list of the SRA accession of interest, use fusera to mount a disk to access the files (shell script)


```R
export DBGAP_NGC="/home/ec2-user/prj_17011.ngc"
export DBGAP_FILETYPE="vcf,vcf_index"
export DBGAP_ACCESSION=`cut -f25 sra.txt | tail -n +2`

mkdir fuseramnt
fusera mount ~/fuseramnt > output.log  2>&1 &
disown %1
```

Step 2: Reduce the number of files by merging them using bcftools. Be careful in the parallelization process since bcftools cannot merge a large number of files at the same time (R code)


```R
vcf = list.files("fuseramnt", pattern = ".vcf", include.dirs = F, recursive = T, full.names = T)
vcf = vcf[-grep(".csi", vcf)]
vcf = split(vcf, rep_len(1:128, length(vcf)))

parallel::mclapply(vcf, function(x) { 
  system(paste0("bcftools merge -m none -O z ",
                paste(x, collapse = ' '),
                " | aws s3 cp - s3://destination/batch1",
                basename(x[1]),
                ".vcf.bgz"))
  }, mc.cores = 64)

print("finish")
```

Step 3: Iterate the process by copying the created files from s3 and merging them together to reduce the number of files


```R
vcf <- system("aws s3 ls s3://destination/batch1 --recursive", intern = T)
unlist <- unlist(lapply(strsplit(vcf, " "), `[`, 4))
vcf <- paste0("aws s3 cp s3://destination/", unlist, " ./vcf/", basename(unlist))

parallel::mclapply(vcf, system, mc.cores = 16)

vcf <- list.files("vcf", pattern = ".vcf.bgz", full.names = T, recursive = T, include.dirs = F)

parallel::mclapply(vcf, function(x) system(paste("bcftools index", x)), mc.cores = 16)

vcf <- split(vcf, rep_len(1:10, length(vcf)))

parallel::mclapply(vcf, function(x) {
  system(paste0("bcftools merge -m none -O z ",
                paste(x, collapse = ' '),
                " | aws s3 cp - s3://destination/batch2/",
                basename(x[1])))
  print(paste("done", basename(x[1])))
}, mc.cores = 12)

print("finish2")
```

Step 4: The last step is the same as the previous one, without the need of parallelization, to create the final merged vcf


```R
file.remove(list.files("vcf", pattern = ".vcf.bgz", full.names = T, include.dirs = F))

vcf <- system("aws s3 ls s3://versmee-etl/COPDbatch2 --recursive", intern = T)
unlist <- unlist(lapply(strsplit(vcf, " "), `[`, 4))
vcf <- paste0("aws s3 cp s3://versmee-etl/", unlist, " ./vcf/", basename(unlist))

parallel::mclapply(vcf, system, mc.cores = 16)

vcf <- list.files("vcf", pattern = ".vcf.bgz", full.names = T, recursive = T, include.dirs = F)

parallel::mclapply(vcf, function(x) system(paste("bcftools index", x)), mc.cores = 16)

system(paste0("bcftools merge -m none -O z ",
                paste(vcf, collapse = ' '),
                " | aws s3 cp - s3://versmee-etl/COPDbatch3/",
               basename(vcf[1])))

print("finish3")
```
