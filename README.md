GenPro
======

Make personal protein databases using next generation sequencing data.

## Installation
### 1. Configure CPAN
Get make if you don't already have it, then configure cpan through the walkthrough:
```
sudo apt install make
cpan
sudo cpan
upgrade
```
(Just keep pressing enter after cpan and it will auto configure)
### 2. Install [local::lib](https://metacpan.org/pod/local::lib) 
```
user@host:~$ cpan
  cpan[1]> look local::lib
  user@host:~/.cpan/build/local-lib-1...$ perl Makefile.PL --bootstrap=~/perl5
  user@host:~/.cpan/build/local-lib-1...$ make && make test && make install
  user@host:~/.cpan/build/local-lib-1...$ exit
  cpan[2]> exit
```
### 3. Install [App::cpanminus](https://metacpan.org/pod/App::cpanminus).
```
curl -L https://cpanmin.us | perl - --sudo App::cpanminus
```
### 4. Clone the GenPro repository
```
sudo apt-get git (if you are starting from scratch)
git clone https://github.com/wingolab/GenPro.git
cd GenPro
```
### 5. Install [Cpanel::JSON::XS](https://github.com/rurban/Cpanel-JSON-XS)
For some reason it fails sometimes to install this dependency so you have to do it manually before installing GenPro
```
(in your top level directory)
git clone https://github.com/rurban/Cpanel-JSON-XS
cd Cpanel-JSON-XS/
perl Makefile.PL
```
### 6. Install GenPro
```
cd GenPro
cpanm GenPro.tar.gz
```
- Install the prebuilt GenPro package in the repository.
```
git clone https://github.com/wingolab/GenPro.git
cd GenPro
cpanm GenPro.tar.gz
```

- Alternatively, untar `GenPro.tar.gz` and install manually. This may require using
`sudo` depending on how Perl is setup.

```
git clone https://github.com/wingolab/GenPro.git
cd GenPro
tar xzvf GenPro.tar.gz
cd GenPro-0.01
perl Makefile.PL
make test
make install    
```

### Dependencies

The minimum version of Perl is v5.10.0. These are all of the dependencies:

    Carp
    Cpanel::JSON::XS
    Data::Dump
    DB_File
    Digest::SHA
    Fcntl
    Getopt::Long
    IO::Uncompress::Gunzip
    Path::Tiny
    Pod::Usage


Most of these come with any standard installation of Perl. To install missing
packages or update old ones, I suggest installing [local::lib](https://metacpan.org/pod/local::lib)
and [App::cpanminus](https://metacpan.org/pod/App::cpanminus), which are two
packages that simplify package installation and do not require `sudo`. After
they are installed you can install/update with:

    cpanm Carp Cpanel::JSON::XS Data::Dump DB_File Digest::SHA Fcntl Getopt::Long \
    IO::Uncompress::Gunzip Path::Tiny

## Usage

- Download genomic data for a particular organism.

```
GenPro_download_ucsc_data.pl -d local_hg38 -g hg38
```

This will perform a dry-run download of hg38 (genome and annotated gene
coordinates). It relies on `rsync` being installed, which should be present on
unix, linux, and OS X by default. In the example, the data will be downloaded 
into `local_hg38` directory, which may be created if it did not already exist. 
`GenPro_download_ucsc_data.pl` will download knownGenes track and the genome of 
the organism by default. By default, `GenPro_download_ucsc_data.pl` is set to a 
dry-run (i.e., no download). Use the `-a` switch to "act", i.e., download the 
data. Take care when using it since it since it will download from a remote 
server. 


- Generate a binary index of the genome for the organism.


  - Use `GenPro_create_db.pl`.
  - There are helper scripts `sh/runall_create_db.sh` that work with SGE to
  build all chromosomes on a cluster with SGE.
    - e.g., `qsub -v USER -v PATH -cwd -t 1-26 runall_create_db.sh <genome>`
  - Alternatively, iterate over all the chromosomes, for hg38:

```
for ((i=1;i<27;i++)); do
  GenPro_create_db.pl -a -g hg38 -c $i --genedir local_hg38/gene --geneset knownGene -f local_hg38/chr -o hg38/idx;
done;
```

- Make reference proteins for the organism.
  - `GenPro_make_refprotdb.pl` uses the indexed binary annotation and creates 
  all proteins for a given chromosome.
  - A helper script `runnall_make_refprot.sh` automates this for SGE.
    - e.g., `qsub -v USER -v PATH -cwd -t 1-26 make_refprot.sh <genome> <reference_proteins_fasta>`


- Make the personal proteins for the sample.
  - this is a 2-step process using `GenPro_make_perprotdb1.pl` and
  `GenPro_make_perprotdb2.pl`
  - `GenPro_make_perprotdb1.pl` creates a per chromosome database of all relevant
  (i.e., missense variants) for all individuals in the snp file.
  - the input file is a 'snp' format; to convert vcf to snp see
  [vcf tools](https://github.com/vcftools) for helper scripts.
  - `GenPro_make_perprotdb2.pl` creates a finished personal protein database for
  a particular sample. The json data encodes just the variant protein information
  while the fasta file encodes the reference and variant proteins.
  - Creating the final db is on a per sample basis, but it can take quite a 
  while. It depends on how many proteins have multiple variants. It is especially 
  slow for proteins with >10 variants. For example, if you have a protein with 
  20 substitutions, there are 20! permutations. By design, all 20! proteins 
  will be considered and only the proteins that contribute unique peptides will
  be retained. Any protien with more than 20 sites will have all variants 
  inserted into the reference protein without performing any permutation. 

