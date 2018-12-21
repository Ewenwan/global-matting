# Global Matting 抠图技术

This project is built by reproducing the global matting algorithm in the CVPR 2011 paper: 

He, Kaiming, et al. "A global sampling method for alpha matting." In CVPR’11, pages 2049–2056, 2011.

    ## 具体实现

    输入:
      1、原图src
     2、掩码图mask(0为背景,128未知区域,255为前景)
    1、expansionOfKnownRegions
     根据前后景图像与周围未知区域的颜色、强度相关性,对图像掩码mask做一定程度的前后景扩散,然后腐蚀。

    2、找到前景掩码和未知区域边界位置,存入:foregroundBoundary
     找到背景掩码和未知区域边界位置,存入:backgroundBoundary

    3、随机生成:foregroundBoundary.size + backgroundBoundary.size个坐标点,如果该坐标位置掩码为前景或者背景,
    则将该点坐标对应放入foregroundBoundary或者backgroundBoundary。

    4、根据foregroundBoundary和backgroundBoundary中存放坐标点对应src图像信息强度,从小到大,对坐标点进行排序。

    5、遍历全图未知区域,分别计算每个未知区域点和foregroundBoundary、backgroundBoundary的最小坐标距离平方差。
    并随机生成(rand() % foregroundBoundary.size(), rand() % backgroundBoundary.size()).全部存入到samples中。 

    6、建立数组二维coords,存入src图像坐标从(0,0)到(h,w)存入。

    7、循环10次:
     7.1、将coords存的坐标数据随机打乱从排。
     7.2、循环coords.size()次,每次取出mask[coords[i].x, coords[i].y]数据,如果mask不是未知区域(128),直接进入下一次循环。
     7.3、未知区域则取出src该点像素与samples该点参数(s1)。
     7.4、以(coords[i].x, coords[i].y)为中心的3x3矩形遍历。
             (1)如果矩形中对应位置mask为128,取出该点samples值(s2)。
             (2)获取到对应前景背景像素值:src(foregroundBoundary[s2.fi]);src(backgroundBoundary[s2.bj]);
             (3)根据当前原地坐标,及取到的前后背景点坐标,论文公式求出alpha,并根据该alpha,求出cost值。
             (4)如果s1中没有存储cost或者cost大于当前cost,则更新当前s1中的fi,bj,cost和alpha值。		   
     7.5、循环src.h*src.w组,每组循环k次。
             (1)随机生成fi,bj。得到F src(foregroundBoundary[fi]), B src(backgroundBoundary[bj])。
             (2)根据每组当前点像素I,以及每次获取到的F,B,计算出alpha,并根据该alpha,求出cost值。
             (3)如果s1中没有存储cost或者cost大于当前cost,则更新当前s1中的fi,bj,cost和alpha值。	   

    8、遍历整个掩码图像,如果mask为未知区域(128),则更新该点像素值为:255 * samples[y][x].alpha。
     进而得到最终的抠图alpha掩码。



## Benchmark

After evaluating the results on the [alpha matting evaluation website](http://alphamatting.com/), this implementation (with matting Laplacian as post-processing) ranks 5th in SAD and 6th in MSE (the original implementation ranks 10th in SAD and 9th in MSE). Running time is less than 1 seconds for an 800x600 image.  Hence, this implementation is one of the highest ranked among the fast matting methods.


## Example

### Code

```c++
#include "globalmatting.h"

// you can get the guided filter implementation
// from https://github.com/atilimcetin/guided-filter
#include "guidedfilter.h"

int main()
{
    cv::Mat image = cv::imread("GT04-image.png", CV_LOAD_IMAGE_COLOR);
    cv::Mat trimap = cv::imread("GT04-trimap.png", CV_LOAD_IMAGE_GRAYSCALE);

    // (optional) exploit the affinity of neighboring pixels to reduce the 
    // size of the unknown region. please refer to the paper
    // 'Shared Sampling for Real-Time Alpha Matting'.
    expansionOfKnownRegions(image, trimap, 9);

    cv::Mat foreground, alpha;
    globalMatting(image, trimap, foreground, alpha);

    // filter the result with fast guided filter
    alpha = guidedFilter(image, alpha, 10, 1e-5);
    for (int x = 0; x < trimap.cols; ++x)
        for (int y = 0; y < trimap.rows; ++y)
        {
            if (trimap.at<uchar>(y, x) == 0)
                alpha.at<uchar>(y, x) = 0;
            else if (trimap.at<uchar>(y, x) == 255)
                alpha.at<uchar>(y, x) = 255;
        }

    cv::imwrite("GT04-alpha.png", alpha);

    return 0;
}
```

### Result

[![Image](http://atilimcetin.com/global-matting/GT04-image_small.png)](http://atilimcetin.com/global-matting/GT04-image.png)
[![Trimap](http://atilimcetin.com/global-matting/GT04-trimap_small.png)](http://atilimcetin.com/global-matting/GT04-trimap.png)
[![Alpha](http://atilimcetin.com/global-matting/GT04-alpha_small.png)](http://atilimcetin.com/global-matting/GT04-alpha.png)


## License

MIT License.


