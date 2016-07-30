##AVT - Automatic Video Region Segmentation and Tracking System
___

####Terms of Use

Copyright (c) 2001 by DVMM Laboratory

Department of Electrical Engineering</br>
Columbia University</br>
Rm 1312 S.W. Mudd, 500 West 120th Street</br>
New York, NY 10027</br>
USA

Di Zhong and Shih-Fu Chang, Columbia University
{dzhong,sfchang}@ee.columbia.edu

####1. SYSTEM OVERVIEW

AVT is an automatic region segmentation and tracking system for general video sources.  It uses feature fusion and motion projection to track salient video regions over long video sequences.


####2. Installation

The system have been developed andtested under Windows platform. There are two packages available: one (AVT_Binary.zip) is the executable binary file only, and one (AVT_Package.zip) is the full package including executable binary, and several other necessary files to show some "click-and-go" demos instructing how to use the tool. Just unzip the package into your specified folder and there is no additional setup is needed. 

Here is a list of the files:

AVT.exe: the executable binary file of AVT.
mpeg2dec.exe: the MPEG-2 (also compatible with MPEG-1) video decoder, needed if the input is MPEG video. 
README.TXT: this helper document
mpeg.cut: the parameter file used in the mpeggo.bat demo
raw.cut: the parameter file used in the rawgo.bat demo
default.par: the default parameter file (optinal for the AVT tool)
MPEGGO.bat: the demo file to show how to parse a mpeg video input
rawgo.bat: the demo file showing how to parse a ppm image sequence input


####3. Usage

Usage: avt {scene.cut} [track.par]

Note:
  - scene cut file is required.
  - mpeg.cut and raw.cut are two example scene cut files.
  - default.par contains the default tracking parameters.


####4. Scene cut file/ Input

The system can take both MPEG(1,2) or raw frames as input. 

NOTE: for raw frames, current version only accept UNIX version PPM format files. The difference between UNIX and Windows version files lies on the explaination of the carriage-return. In UNIX platform, "carriage-return" will be translated as one ASCII symbol "0X0A". In Windows, instead, two ASCII symbols "0X0D, 0X0A". Please make sure you are providing files with correct format.

----mpeg.cut------
MPEG                              -> Magic
/proj/oscar1/VQ/AVT/TMP           -> Output directory
/proj/oscar1/VQ/AVT/akiyo.mpg     -> MPEG file name
2                                 -> Number of shots to track
10 15                             -> first shot, start & end frame
30 34                             -> second shot, start & end frame

----raw.cut------ 
RAW                                               -> Magic
2                                                 -> Number of shots to track
/proj/oscar1/VQ/AVT/TMP/akiyo_10/f%.5d.ppm 10 15  -> first shot directory,start&end frame
/proj/oscar1/VQ/AVT/TMP/akiyo_30/f%.5d.ppm 30 34  -> first shot directory,start&end frame


####5. Output

For each frame, avt generates following output files:

  - .seg file: segmented regions drawn in PPM format with mean colors
  - .oif file: segmented regions with their basic features (see next section for detail)


####6. OIF file - Data structure and I/O functions
```
//
//ObjectFile is an list of ObjectInfo
class ObjectFile : public DArray<ObjectInfo> {
public:

  int available_id ; //used in tracking process to insure unique object ID
  int frame_w, frame_h ;
  //
  //Note: following features are not computed at this version
  float affine[6] ; //global motion paramerter
  float mv[2] ; //average motion vector
  char pan ; //-1: no panning; 0-7: panning direction
  char zoom ; //-1: no zooming; 0: zoom in; 1: zoom out ;

  //
  //Constructor
  ObjectFile(char *) {
    ifstream in(fname) ;

    if(!in) {
      cerr<<"Error: can not open OIF "<<fname<<endl ;
      available_id=0 ;
      return ;
    }

    //
    // Readin head information
    char buf[MAXLEN] ;
    int n ;
    in.read(&n, sizeof(int)) ;
    in.read(&available_id, sizeof(int)) ;
    in.read(&frame_w, sizeof(int)) ;
    in.read(&frame_h, sizeof(int)) ;
    in.read(&affine, sizeof(float)*6) ;
    in.read(&mv, sizeof(float)*2) ;
    in.read(&pan, 1) ;
    in.read(&zoom, 1) ;

    //
    // Readin each object
    for(int i=0; i<n; i++){
      ObjectInfo *obj=new ObjectInfo(in) ;
      add(obj) ;
    }

    in.close() ;
  };

  // Binary format output
	void save(char *fname) {
		ofstream out(fname) ;
		if(!out) {
			cerr<<"Error: can not creat "<<fname<<endl ;
			return ;
		}

		int cnum=number() ;
		out.write(&cnum, sizeof(int) ) ;
		out.write(&available_id, sizeof(int) ) ;
		out.write(&frame_w, sizeof(int) ) ;
		out.write(&frame_h, sizeof(int) ) ;
		out.write(affine, 6*sizeof(float)) ;
		out.write(mv, 2*sizeof(float)) ;
		out.write(&pan, 1) ;
		out.write(&zoom, 1) ;
		for(int i=0; i<number(); i++)
			(*this)[i].save(out) ;

		out.close() ;
  }
} ;


//
//ObjectInfo contains basic features of a segmented region
class ObjectInfo {
public:
  int id ;  //unique ID for this object

  int pixel_num ;
  int ul[2], lr[2] ; // Upper_left and lower_right
  double luv[3] ;  //representative color in L*u*v* space
  int cs ;  //0 for Luv space (always 0 at this version)
  float mv[2] ; //average motion vector
  float affine[6] ; //Affine model parameters
  char  bg ; //1:Background; 0: Foreground **NOT** valid at this version

  unsigned char *mask ; // size=(y1-y0+1)*(x1-x0+1), left_right/top_down
  DArray<int> connected_obj_ids ;

	ObjectInfo(istream& in) {
		int cnum ;
		char tmp ;

		in.read((char *)&id, sizeof(int)) ;
		in.read((char *)&cnum, sizeof(int)) ;
		in.read((char *)&pixel_num, sizeof(int)) ;
		in.read((char *)ul, 2*sizeof(int)) ;
		in.read((char *)lr, 2*sizeof(int)) ;
		in.read(&tmp, 1) ;
		cs=(int)tmp ;
		in.read((char *)luv, 3*sizeof(double)) ;
		in.read((char *)mv, 2*sizeof(float)) ;
		in.read((char *)affine, 6*sizeof(float)) ;
		in.read(&bg, 1) ;

		if(pixel_num) {
			int s=width()*height() ;
			if(s) {
				mask=new unsigned char[s] ;
				in.read((char *)mask, s) ;
			}
		}else {
			mask=NULL ;
		}

    connected_obj_ids=new int[cnum] ;
		for(int i=0; i<cnum; i++){
      int cid ;
			o.read((char *)&cid, sizeof(int)) ;
      connected_obj_ids[i]=cid ;
		}
	}

	void save(ostream& o) {
		int cnum=connected_obj_ids.number() ;

		o.write((char *)&id, sizeof(int)) ;
		o.write((char *)&cnum, sizeof(int)) ;
		o.write((char *)&pixel_num, sizeof(int)) ;
		o.write((char *)ul, 2*sizeof(int)) ;
		o.write((char *)lr, 2*sizeof(int)) ;
		char tmp=(int)cs ;
		o.write(&tmp, 1) ;
		o.write((char *)luv, 3*sizeof(double)) ;
		o.write((char *)mv, 2*sizeof(float)) ;
		o.write((char *)affine, 6*sizeof(float)) ;
		o.write(&bg, 1) ;

		int s=width()*height() ;
		if(pixel_num==0) s=0 ;
		if(s) o.write((char *)mask, s) ;

		for(int i=0; i<cnum; i++){
      s=connected_obj_ids[i] ;
			o.write((char *)&s, sizeof(int)) ;
		}
	}
} ;
```