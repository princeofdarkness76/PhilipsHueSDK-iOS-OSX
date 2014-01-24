#Application Design note - Color conversion


Current Philips lights have a color gamut defined by 3 points, making it a triangle.


    Red: 0.675, 0.322

    Red: 0.704, 0.296

    Red: 1.0, 0
    
    float green = (green > 0.04045f) ? pow((green + 0.055f) / (1.0f + 0.055f), 2.4f) : (green / 12.92f);
    float blue = (blue > 0.04045f) ? pow((blue + 0.055f) / (1.0f + 0.055f), 2.4f) : (blue / 12.92f);

    float Y = red * 0.234327f + green * 0.743075f + blue * 0.022598f;
    float Z = red * 0.0000000f + green * 0.053077f + blue * 1.035763f;
    
    
6. Calculate the closest point on the color gamut triangle and use that as xy value
The Y value indicates the brightness of the converted color.

		
    float x = x; // the given x value
    float y = y; // the given y value
    float z = 1.0f - x - y; 
    float Y = brightness; // The given brightness value
    float X = (Y / y) * x;  
    float Z = (Y / y) * z;

	
    float r = X * 1.612f - Y * 0.203f - Z * 0.302f;
    float g = -X * 0.509f + Y * 1.412f + Z * 0.066f;
    float b = X * 0.026f - Y * 0.072f + Z * 0.962f;
    
    g = g <= 0.0031308f ? 12.92f * g : (1.0f + 0.055f) * pow(g, (1.0f / 2.4f)) - 0.055f;
    b = b <= 0.0031308f ? 12.92f * b : (1.0f + 0.055f) * pow(b, (1.0f / 2.4f)) - 0.055f;
 


##Code Examples
The following code is an extract from the relevant methods in the iOS SDK. It is provided on an as is basis to help you create your own versions of the color conversion utilities.
Note that the UIColor class contains a method to obtain Hue/Saturation values http://developer.apple.com/library/ios/#documentation/UIKit/Reference/UIColor_Class/Reference/Reference.html

    + (UIColor *)colorFromXY:(CGPoint)xy forModel:(NSString*)model {        
        NSArray *colorPoints = [self colorPointsForModel:model];
        BOOL inReachOfLamps = [self checkPointInLampsReach:xy withColorPoints:colorPoints];
        
        if (!inReachOfLamps) {
            //It seems the colour is out of reach
            //let's find the closest colour we can produce with our lamp and send this XY value out.
            
            //Find the closest point on each line in the triangle.
            CGPoint pAB =[self getClosestPointToPoints:[self getPointFromValue:[colorPoints objectAtIndex:cptRED]] point2:[self getPointFromValue:[colorPoints objectAtIndex:cptGREEN]] point3:xy];
            CGPoint pAC = [self getClosestPointToPoints:[self getPointFromValue:[colorPoints objectAtIndex:cptBLUE]] point2:[self getPointFromValue:[colorPoints objectAtIndex:cptRED]] point3:xy];
            CGPoint pBC = [self getClosestPointToPoints:[self getPointFromValue:[colorPoints objectAtIndex:cptGREEN]] point2:[self getPointFromValue:[colorPoints objectAtIndex:cptBLUE]] point3:xy];
            
            //Get the distances per point and see which point is closer to our Point.
            float dAB = [self getDistanceBetweenTwoPoints:xy point2:pAB];
            float dAC = [self getDistanceBetweenTwoPoints:xy point2:pAC];
            float dBC = [self getDistanceBetweenTwoPoints:xy point2:pBC];
            
            float lowest = dAB;
            CGPoint closestPoint = pAB;
            
            if (dAC < lowest) {
                lowest = dAC;
                closestPoint = pAC;
            }
            if (dBC < lowest) {
                lowest = dBC;
                closestPoint = pBC;
            }
            
            //Change the xy value to a value which is within the reach of the lamp.
            xy.x = closestPoint.x;
            xy.y = closestPoint.y;
        }
        
        float x = xy.x;
        float y = xy.y;
        float z = 1.0f - x - y;
        
        float Y = 1.0f;
        float X = (Y / y) * x;
        float Z = (Y / y) * z;
        
        // sRGB D65 conversion
        float r = X  * 3.2406f - Y * 1.5372f - Z * 0.4986f;
        float g = -X * 0.9689f + Y * 1.8758f + Z * 0.0415f;
        float b = X  * 0.0557f - Y * 0.2040f + Z * 1.0570f;
        
        if (r > b && r > g && r > 1.0f) {
            // red is too big
            g = g / r;
            b = b / r;
            r = 1.0f;
        }
        else if (g > b && g > r && g > 1.0f) {
            // green is too big
            r = r / g;
            b = b / g;
            g = 1.0f;
        }
        else if (b > r && b > g && b > 1.0f) {
            // blue is too big
            r = r / b;
            g = g / b;
            b = 1.0f;
        }
        
        // Apply gamma correction
        r = r <= 0.0031308f ? 12.92f * r : (1.0f + 0.055f) * pow(r, (1.0f / 2.4f)) - 0.055f;
        g = g <= 0.0031308f ? 12.92f * g : (1.0f + 0.055f) * pow(g, (1.0f / 2.4f)) - 0.055f;
        b = b <= 0.0031308f ? 12.92f * b : (1.0f + 0.055f) * pow(b, (1.0f / 2.4f)) - 0.055f;
        
        if (r > b && r > g) {
            // red is biggest
            if (r > 1.0f) {
                g = g / r;
                b = b / r;
                r = 1.0f;
            }
        }
        else if (g > b && g > r) {
            // green is biggest
            if (g > 1.0f) {
                r = r / g;
                b = b / g;
                g = 1.0f;
            }
        }
        else if (b > r && b > g) {
            // blue is biggest
            if (b > 1.0f) {
                r = r / b;
                g = g / b;
                b = 1.0f;
            }
        }

        return [UIColor colorWithRed:r green:g blue:b alpha:1.0f];
    }
    
    + (NSArray *)colorPointsForModel:(NSString*)model {
        NSMutableArray *colorPoints = [NSMutableArray array];
        
        NSArray *hueBulbs = [NSArray arrayWithObjects:@"LCT001" /* Hue A19 */,
                             @"LCT002" /* Hue BR30 */,
                             @"LCT003" /* Hue GU10 */, nil];
        NSArray *livingColors = [NSArray arrayWithObjects:  @"LLC001" /* Monet, Renoir, Mondriaan (gen II) */,
                                 @"LLC005" /* Bloom (gen II) */,
                                 @"LLC006" /* Iris (gen III) */,
                                 @"LLC007" /* Bloom, Aura (gen III) */,
                                 @"LLC011" /* Hue Bloom */,
                                 @"LLC012" /* Hue Bloom */,
                                 @"LLC013" /* Storylight */,
                                 @"LST001" /* Light Strips */, nil];
        if ([hueBulbs containsObject:model]) {
            // Hue bulbs color gamut triangle
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(0.674F, 0.322F)]];     // Red
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(0.408F, 0.517F)]];     // Green
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(0.168F, 0.041F)]];     // Blue
            
        }
        else if ([livingColors containsObject:model]) {
            // LivingColors color gamut triangle
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(0.703F, 0.296F)]];     // Red
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(0.214F, 0.709F)]];     // Green
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(0.139F, 0.081F)]];     // Blue
        }
        else {
            // Default construct triangle wich contains all values
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(1.0F, 0.0F)]];         // Red
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(0.0F, 1.0F)]];         // Green
            [colorPoints addObject:[self getValueFromPoint:CGPointMake(0.0F, 0.0F)]];         // Blue
        }
        
        return colorPoints;
    }
    
    + (CGPoint)calculateXY:(UIColor *)color forModel:(NSString*)model {
    	CGColorRef cgColor = [color CGColor];
            
        const CGFloat *components = CGColorGetComponents(cgColor);
        long numberOfComponents = CGColorGetNumberOfComponents(cgColor);
            
        // Default to white
        CGFloat red = 1.0f;
        CGFloat green = 1.0f;
        CGFloat blue = 1.0f;
           
        if (numberOfComponents == 4) {
            // Full color
            red = components[0];
            green = components[1];
            blue = components[2];
        }
        else if (numberOfComponents == 2) {
            // Greyscale color
            red = green = blue = components[0];
        }
            
        // Apply gamma correction
        float r = (red   > 0.04045f) ? pow((red   + 0.055f) / (1.0f + 0.055f), 2.4f) : (red   / 12.92f);
        float g = (green > 0.04045f) ? pow((green + 0.055f) / (1.0f + 0.055f), 2.4f) : (green / 12.92f);
        float b = (blue  > 0.04045f) ? pow((blue  + 0.055f) / (1.0f + 0.055f), 2.4f) : (blue  / 12.92f);
            
        // Wide gamut conversion D65
        float X = r * 0.649926f + g * 0.103455f + b * 0.197109f;
        float Y = r * 0.234327f + g * 0.743075f + b * 0.022598f;
        float Z = r * 0.0000000f + g * 0.053077f + b * 1.035763f;
            
        float cx = X / (X + Y + Z);
        float cy = Y / (X + Y + Z);
           
        if (isnan(cx)) {
            cx = 0.0f;
        }
            
        if (isnan(cy)) {
            cy = 0.0f;
        }
            
        //Check if the given XY value is within the colourreach of our lamps.
           
        CGPoint xyPoint =  CGPointMake(cx,cy);
        NSArray *colorPoints = [self colorPointsForModel:model];
        BOOL inReachOfLamps = [self checkPointInLampsReach:xyPoint withColorPoints:colorPoints];
            
        if (!inReachOfLamps) {
       	    //It seems the colour is out of reach
            //let's find the closest colour we can produce with our lamp and send this XY value out.
                
            //Find the closest point on each line in the triangle.
            CGPoint pAB =[self getClosestPointToPoints:[self getPointFromValue:[colorPoints objectAtIndex:cptRED]] point2:[self getPointFromValue:[colorPoints objectAtIndex:cptGREEN]] point3:xyPoint];
            CGPoint pAC = [self getClosestPointToPoints:[self getPointFromValue:[colorPoints objectAtIndex:cptBLUE]] point2:[self getPointFromValue:[colorPoints objectAtIndex:cptRED]] point3:xyPoint];
            CGPoint pBC = [self getClosestPointToPoints:[self getPointFromValue:[colorPoints objectAtIndex:cptGREEN]] point2:[self getPointFromValue:[colorPoints objectAtIndex:cptBLUE]] point3:xyPoint];
                
            //Get the distances per point and see which point is closer to our Point.
            float dAB = [self getDistanceBetweenTwoPoints:xyPoint point2:pAB];
            float dAC = [self getDistanceBetweenTwoPoints:xyPoint point2:pAC];
            float dBC = [self getDistanceBetweenTwoPoints:xyPoint point2:pBC];
                
            float lowest = dAB;
            CGPoint closestPoint = pAB;
                
            if (dAC < lowest) {
                lowest = dAC;
                closestPoint = pAC;
            }
            if (dBC < lowest) {
                lowest = dBC;
                closestPoint = pBC;
            }
              
            //Change the xy value to a value which is within the reach of the lamp.
            cx = closestPoint.x;
            cy = closestPoint.y;
        }
           
        return CGPointMake(cx, cy);
    }
        
    /**
     * Calculates crossProduct of two 2D vectors / points.
     *
     * @param p1 first point used as vector
     * @param p2 second point used as vector
     * @return crossProduct of vectors
     */
    + (float)crossProduct:(CGPoint)p1 point2:(CGPoint)p2 {
        return (p1.x * p2.y - p1.y * p2.x);
    }
        
    /**
     * Find the closest point on a line.
     * This point will be within reach of the lamp.
     *
     * @param A the point where the line starts
     * @param B the point where the line ends
     * @param P the point which is close to a line.
     * @return the point which is on the line.
     */
    + (CGPoint)getClosestPointToPoints:(CGPoint)A point2:(CGPoint)B point3:(CGPoint)P {
        CGPoint AP = CGPointMake(P.x - A.x, P.y - A.y);
        CGPoint AB = CGPointMake(B.x - A.x, B.y - A.y);
        float ab2 = AB.x * AB.x + AB.y * AB.y;
        float ap_ab = AP.x * AB.x + AP.y * AB.y;
            
        float t = ap_ab / ab2;
          
        if (t < 0.0f) {
            t = 0.0f;
        }
        else if (t > 1.0f) {
            t = 1.0f;
        }
            
        CGPoint newPoint = CGPointMake(A.x + AB.x * t, A.y + AB.y * t);
        return newPoint;
    }
        
    /**
     * Find the distance between two points.
     *
     * @param one
     * @param two
     * @return the distance between point one and two
     */
    + (float)getDistanceBetweenTwoPoints:(CGPoint)one point2:(CGPoint)two {
        float dx = one.x - two.x; // horizontal difference
        float dy = one.y - two.y; // vertical difference
        float dist = sqrt(dx * dx + dy * dy);
            
        return dist;
    }
        
    /**
     * Method to see if the given XY value is within the reach of the lamps.
     *
     * @param p the point containing the X,Y value
     * @return true if within reach, false otherwise.
     */
    + (BOOL)checkPointInLampsReach:(CGPoint)p withColorPoints:(NSArray*)colorPoints {
        CGPoint red =   [self getPointFromValue:[colorPoints objectAtIndex:cptRED]];
        CGPoint green = [self getPointFromValue:[colorPoints objectAtIndex:cptGREEN]];
        CGPoint blue =  [self getPointFromValue:[colorPoints objectAtIndex:cptBLUE]];
           
        CGPoint v1 = CGPointMake(green.x - red.x, green.y - red.y);
        CGPoint v2 = CGPointMake(blue.x - red.x, blue.y - red.y);
            
        CGPoint q = CGPointMake(p.x - red.x, p.y - red.y);
            
        float s = [self crossProduct:q point2:v2] / [self crossProduct:v1 point2:v2];
        float t = [self crossProduct:v1 point2:q] / [self crossProduct:v1 point2:v2];
            
        if ( (s >= 0.0f) && (t >= 0.0f) && (s + t <= 1.0f)) {
            return true;
        }
        else {
            return false;
        }
    }

##Further Information
The following links provide further useful related information

sRGB:

http://en.wikipedia.org/wiki/Srgb  

A Review of RGB Color Spaces:
 
http://www.babelcolor.com/download/A%20review%20of%20RGB%20color%20spaces.pdf

Useful color equations:
http://www.brucelindbloom.com/




