# Median-of-all--maximum-payments-over-all-my-clients

    Median of all- maximum payments over all my clients

    I need to calculate the median of the maximum payments over al my clients

    R/IML to the rescue

    github
    https://gist.github.com/rogerjdeangelis/6939a86b08c29068678141c008e98c22

    related to
    https://goo.gl/cRXgeG
    https://communities.sas.com/t5/Base-SAS-Programming/question-about-coding-array-for-money-and-choosing-highest/m-p/416354


    INPUT
    =====
    SD1.HAVE total obs=4                          |    RULES
                                                  |  ========
        ID       PAYMENT1    PAYMENT2    PAYMENT3 |   Maximums
                                                  |
      Person1       100           .           .   |     100
      Person2         .         450         400   |     450
      Person3         .         450           .   |     450
      Person4       500           .         750   |     750

                                                     Median = (450 + 450)/2 = 450


    WORKING CODE
    ============
       R
       ==
        want<-median(apply(have,1,max,na.rm=TRUE));

       SAS
       ===

        PHASE ONE COMPILE TIME DOSUBL

                data _null_;
                   set sd1.have nobs=numobs;
                   call symputx('numobs',numobs);
                run;quit;

                array pays[&numobs.] pay1-pay&numobs.;
                retain pay1-pay&numobs;

        MAINLINE

          set sd1.have end=dne;
          pays[_n_]=max(of payment:);  * load all records into array;

          if dne then do;

             call sortn(of pays[*]);   * done so order maximumsl

             if mod(&numobs.,2)=0 then   * even number of values;
                median=(pays[floor(&numobs./2)] + pays[ceil(&numobs./2)])/2;
             else
                median=pays[(&numobs.+1)/2]; * odd number;
             output;
          end;

    OUTPUT
    ======
       R
       %put &=sasMacroVariableFromR;

       sasMacroVariableFromR = 450

      SAS

      WORK.WANT total obs=1

      Obs    PAY1    PAY2    PAY3    PAY4    MEDIAN
       1      100     450     450     750      450


    *                _              _       _
     _ __ ___   __ _| | _____    __| | __ _| |_ __ _
    | '_ ` _ \ / _` | |/ / _ \  / _` |/ _` | __/ _` |
    | | | | | | (_| |   <  __/ | (_| | (_| | || (_| |
    |_| |_| |_|\__,_|_|\_\___|  \__,_|\__,_|\__\__,_|

    ;

    options validvarname=upcase;
    libname sd1 "d:/sd1";
    data sd1.have;
      input id$ Payment1 Payment2 Payment3;
    cards;
    Person1 100 . .
    Person2 . 450 400
    Person3 . 450 .
    Person4 500 . 750
    ;;;;
    run;quit;

    *                          _       _   _
     ___  __ _ ___   ___  ___ | |_   _| |_(_) ___  _ __
    / __|/ _` / __| / __|/ _ \| | | | | __| |/ _ \| '_ \
    \__ \ (_| \__ \ \__ \ (_) | | |_| | |_| | (_) | | | |
    |___/\__,_|___/ |___/\___/|_|\__,_|\__|_|\___/|_| |_|

    ;

    * array solution;
    data want;
      * size of array;
      if _n_=0 then do;
         %let rc=%sysfunc(dosubl('
            data _null_;
               set sd1.have nobs=numobs;
               call symputx('numobs',numobs);
            run;quit;
         '));
         array pays[&numobs.] pay1-pay&numobs.;
         retain pay1-pay&numobs;
      end;
      set sd1.have end=dne;
      pays[_n_]=max(of payment:);
      if dne then do;
         call sortn(of pays[*]);
         if mod(&numobs.,2)=0 then
            median=(pays[floor(&numobs./2)] + pays[ceil(&numobs./2)])/2;
         else
            median=pays[(&numobs.+1)/2];
         output;
      end;
      drop payment: id;
    run;quit;

    Up to 40 obs from want total obs=1

    Obs    PAY1    PAY2    PAY3    PAY4    MEDIAN

     1      100     450     450     750      450


    *____              _       _   _
    |  _ \   ___  ___ | |_   _| |_(_) ___  _ __
    | |_) | / __|/ _ \| | | | | __| |/ _ \| '_ \
    |  _ <  \__ \ (_) | | |_| | |_| | (_) | | | |
    |_| \_\ |___/\___/|_|\__,_|\__|_|\___/|_| |_|

    ;

    %symdel sasmacrovariable;

    %utl_submit_r64(
      '
       library(haven);
       have<-read_sas("d:/sd1/have.sas7bdat")[,-1];
       have;
       want<-median(apply(have,1,max,na.rm=TRUE));
       want<-format(want);
       writeClipboard(want);
      '
     ,returnvar=sasMacroVariableFromR
    );

    %out &=sasMacroVariableFromR

    *
     _ __ ___   __ _  ___ _ __ ___
    | '_ ` _ \ / _` |/ __| '__/ _ \
    | | | | | | (_| | (__| | | (_) |
    |_| |_| |_|\__,_|\___|_|  \___/

    ;

    %macro utl_submit_r64(
          pgmx
         ,returnVar=N           /* set to Y if you want a return SAS macro variable from python */
         )/des="Semi colon separated set of R commands - drop down to R";
      * write the program to a temporary file;
      filename r_pgm "d:/txt/r_pgm.txt" lrecl=32766 recfm=v;
      data _null_;
        length pgm $32756;
        file r_pgm;
        pgm=&pgmx;
        put pgm;
        putlog pgm;
      run;
      %let __loc=%sysfunc(pathname(r_pgm));
      * pipe file through R;
      filename rut pipe "c:\Progra~1\R\R-3.3.1\bin\x64\R.exe --vanilla --quiet --no-save < &__loc";
      data _null_;
        file print;
        infile rut recfm=v lrecl=32756;
        input;
        put _infile_;
        putlog _infile_;
      run;
      filename rut clear;
      filename r_pgm clear;

      * use the clipboard to create macro variable;
      %if %upcase(%substr(&returnVar.,1,1)) ne N %then %do;
        filename clp clipbrd ;
        data _null_;
         length txt $200;
         infile clp;
         input;
         putlog "macro variable &returnVar = " _infile_;
         call symputx("&returnVar.",_infile_,"G");
        run;quit;
      %end;

    %mend utl_submit_r64;


