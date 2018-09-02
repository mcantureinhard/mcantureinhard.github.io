---
layout: post
title:  "Minimalistic arduino communication: Led graph bar and image processing - PART 1"
date:   2018-08-19 21:00:00 +0200
categories: [java,image-processing, android, microcontroller, led]
---
I have been using the esp8266 for a while and exploring the use of least possible add-ons(to minimize costs) while interacting with it. The esp8266 has different modes of operation, the most common being access point and client. A typical usage scenario while using the esp8266 would be to set it in access point mode, no password set, connect to it and submit a form with the ssid and password for the network it should join. In my scenario, the esp8266 works only in access point mode and the user connects to it. I could use a fixed password, but I want a randomly generated password each time the esp8266 turns on. I could opt to use a display, but this will probably be more expensive.

I started to think about different ways to communicate and came up with a couple of ideas. The first of these is to use a blinking led graph bar to send the password to a phone via the camera. In theory, this seems reasonable, most phones have cameras capable of recording video at 30 fps. There are 2 parts required to get this to work, image processing to identify the state of the leds and a microcontroller with blinking leds. Since encoding characters into on/off leds is simple, I decided to get started with the image processing. Having an android phone, I'll be using Java.

My first implementation was very basic. Loop through the image, have an additional array of visited points and when finding a color match from the rgb colors array do a graph like traveral. In this traversal I go in all directions iteratively by using a queue, checking if the point has been visited or not. I also added a local visited array when doing the traversal to avoid repeated traversals without modifying the global visited array, although this may not be necessary. I tested two ways of calculating the color distance; Manhattan distance and Euclidean distance. By avoiding Math.pow and Math.sqrt I thought I would be able to improve the performance of my blob detector, but my results when running on my machine showed me otherwise. The cold runtime for one color was around 40ms, while that for 3 colors was around 80ms. At 80ms the best I can do is 12.5 FPS, so this one is not good enough.

Sample image:
![Sample Image]({{ "/assets/images/led_graph_bar/sample.png" }})

Result (Matching colors are [0,255,195],[0,0,255],[255,0,0]):

![Sample Image]({{ "/assets/images/led_graph_bar/BlobFinder.png" }})

I have helper class ImageHandler, which has some code to draw the red rectangles on the found blobs.

My first implementation of blob detection is the following:
{% highlight java %}
public class BlobFinder {
    private int minSize = 50;
    private double maxColorDistance = 50;
    private int[] rgbColors;
    private int background = 0xffffff;
    private ImageHandler handler;
    public BlobFinder(ImageHandler handler){
        this.handler = handler;
    }
    public BlobFinder(ImageHandler handler, int[][] rgbColors){
        if(rgbColors.length == 0 || rgbColors[0].length != 3){
            System.out.println("Error");
            return;
        }
        this.rgbColors = new int[rgbColors.length];
        int count = 0;
        for(int[] colorArr: rgbColors){
            this.rgbColors[count] = rgbArrayToInt(colorArr);
            count++;
        }

        this.handler = handler;
    }
    public BlobFinder(ImageHandler handler, int[] rgbColors){
        this.rgbColors = rgbColors;
        this.handler = handler;
    }
    public BlobFinder(ImageHandler handler, int[] rgbColors, int minSize){
        this.rgbColors = rgbColors;
        this.handler = handler;
        this.minSize = minSize;
    }
    public ArrayList<Blob> getBlobs(){
        ArrayList<Blob> blobs = new ArrayList<Blob>();
        BufferedImage img = handler.getInputImage();
        int[][] visited = new int[img.getHeight()][img.getWidth()];
        for(int i = 0; i < img.getHeight();i++){      //y
            for(int j = 0;j<img.getWidth();j++){  //x
                int pixelVisited = visited[i][j];
                if(pixelVisited == 1){
                    continue;
                }
                int pixel = img.getRGB(j,i);
                if(pixel == -1){
                    visited[i][j] = 1;
                    continue;
                }
                double bgDistance = rgbDistance(background,pixel);
                if(bgDistance <= 100){
                    visited[i][j] = 1;
                    continue;
                }
                int foundColor = background;
                if(rgbColors != null){
                    double minDistance = maxColorDistance + 1;
                    for(int color: rgbColors){
                        double temp = rgbDistance(color, pixel);
                        if(temp < minDistance){
                            foundColor = color;
                            minDistance = temp;
                        }
                    }
                    if(minDistance > maxColorDistance){
                        visited[i][j] = 1;
                        continue;
                    }
                }
                Blob newBlob = getBlob(img, visited,j, i, foundColor);
                if(newBlob != null){
                    blobs.add(newBlob);
                }
            }
        }
        return blobs;
    }
    private Blob getBlob(BufferedImage img, int[][] visited, int x, int y, int matchColor) {
        int max_x = x;
        int min_x = x;
        int max_y = y;
        int min_y = y;
        Random rand = new Random();
        int rr = rand.nextInt(255);
        int rg = rand.nextInt(255);
        int rb = rand.nextInt(255);
        int base_color;
        int alpha = 0;
        if(matchColor == background) {
            base_color = img.getRGB(x,y);
        } else {
            base_color = matchColor;
        }
        Queue <Pair <Integer, Integer> > queue = new LinkedList<>();
        queue.add(new Pair(x,y));
        int[][] visitedBlob = new int[img.getHeight()][img.getWidth()];
        int size = 0;
        while(queue.peek() != null){
            Pair current = queue.remove();
            int \_x = (int)current.getLeft();
            int \_y = (int)current.getRight();
            if(visitedBlob[_y][_x] == 1 || visited[_y][_x] == 1){
                continue;
            }
            visitedBlob[_y][_x] = 1;
            int pixel = img.getRGB(\_x,\_y);
            if(rgbDistance(base_color,pixel) > maxColorDistance){
                continue;
            }
            size++;
            visited[_y][_x] = 1;
            alpha = (alpha + 1)%255;
            if(\_x < min_x){
                min_x = \_x;
            }
            if(\_x > max_x){
                max_x = \_x;
            }
            if(\_y < min_y){
                min_y = \_y;
            }
            if(\_y > max_y){
                max_y = \_y;
            }
            if(\_x>0 && visitedBlob[_y][_x - 1] != 1){
                queue.add(new Pair((\_x - 1),\_y));
            }
            if(\_x < (img.getWidth() - 1) && visitedBlob[_y][_x + 1] != 1){
                queue.add(new Pair((\_x + 1),\_y));
            }
            if(\_y > 0 && visitedBlob[\_y - 1][\_x] != 1){
                queue.add(new Pair(\_x ,(\_y - 1)));
            }
            if(\_y < (img.getHeight() - 1) && visitedBlob[_y + 1][_x] != 1){
                queue.add(new Pair(\_x,\_y + 1));
            }
        }
        if(size < minSize){
            return null;
        }
        Blob blob = new Blob();
        blob.setRgb(intToRgbArray(base_color));
        int posx = (max_x-min_x)/2 + min_x;
        int posy = (max_y-min_y)/2 + min_y;
        blob.setPosition(posx, posy);
        blob.setHeight(max_y - min_y);
        blob.setWidth(max_x - min_x);
        return blob;
    }
    private int rgbArrayToInt(int[] rgb){
        int color = (rgb[0] << 16) | (rgb[1] << 8) | rgb[2];
        return color;
    }
    private int[] intToRgbArray(int rgb){
        int  red   = (rgb & 0x00ff0000) >> 16;
        int  green = (rgb & 0x0000ff00) >> 8;
        int  blue  =  rgb & 0x000000ff;
        int[] rgbArr = {red, green, blue};
        return rgbArr;
    }
    private double manhattanDistance(int a0, int a1, int b0, int b1, int c0, int c1){
        double da = Math.abs(a0 - a1);
        double db = Math.abs(b0 - b1);
        double dc = Math.abs(c0 - c1);
        return da+db+dc;
    }
    private double rgbDistance(int a, int b){
        int  red   = (a & 0x00ff0000) >> 16;
        int  green = (a & 0x0000ff00) >> 8;
        int  blue  =  a & 0x000000ff;
        int  red_2   = (b & 0x00ff0000) >> 16;
        int  green_2 = (b & 0x0000ff00) >> 8;
        int  blue_2  =  b & 0x000000ff;

        double da = Math.abs(red -red_2);
        double db = Math.abs(green - green_2);
        double dc = Math.abs(blue - blue_2);
        return da+db+dc;
    }
}
{% endhighlight %}

Having this take 80ms for three colors led me into trying something alternative at the time. It has been a long time since I have used Java more than the usual basics and I found out about RecursiveTask. I decided to give it a try. The idea was to split the image into four parts as subtaks until I got to a pixel level. Then start joining the results based on certain conditions. If at least three of the parts were a match, I would continue to return 1. Otherwise I would build pieces of blobs that I would then stick together at the end. This did not go well. I ended up with multiple java.util.concurrent.CancellationException. I went for the traditional fork/join and I doubt using Future would have helped in any way. This test was in the end a terrible failure.

Afterwards I decided to try splitting the image processing into a more reasonable number of threads, four. Although this did improve the runtime a bit, it wasn't enough. By this time I had already dug into getting something running on Android. I had already tested the camera2 API and I decided to make a few more changes for my new class. I decided to receive the data as a one dimensional array (I decded to not care about the padding and other image stuff, for the moment), I added splitting into multiple threads on a row level (processing in order in the array could potentially help reduce the number of cache faults and thus increase performance), I added a minimum size for width and height of a blob, I process rows first, encoding 64 columns into one long to save memory, I added counters for row level and column level matches to allow skipping rows/columns without any possible blobs when I process the columns, I skip one row and one column at a time to only process number_of_pixels/4 (I later realized that with the introduction of the minimum size I could have skipped many more columns if there was no match) and I keep track of the matched color to avoid looping colors when I am checking for continuity of my last match. For a cold run, the runtime with 3 colors is 20ms on a single thread. When I ran my code with two threads, cold runtime was down to 15ms, but I realized I have some bugs. Since the single threaded performance would allow me 50 FPS I decided to leave it as is.

Result (Matching colors are [0,255,195],[255,0,192],[0,24,255]):

![Sample Image]({{ "/assets/images/led_graph_bar/foundSplit.png" }})

The code:

{% highlight java %}
public class FastBlobFinderSkip {
    int[] bytes;
    int width;
    int height;
    int pixelStride;
    int[][] colors;
    int minSize = 32;
    int masklen;
    long maskLeft;
    int pixels_w = 2;
    int pixels_h = 2;
    int column_width = 128;

    public FastBlobFinderSkip(int[] bytes, int width, int height, int pixelStride, int[][] colors) {
        this.bytes = bytes;
        this.width = width;
        this.height = height;
        this.pixelStride = pixelStride;
        this.colors = colors;
        this.masklen = Long.SIZE;
        this.maskLeft = 1L << (this.masklen -1);
    }
    public ArrayList<Blob> find(int split){
        Thread[] threads = new Thread[split];
        RowCheckerThread[] checkers = new RowCheckerThread[split];
        int size = height / split;
        for(int i = 0; i < split;i++){
            int start = i*size;
            int end = start + size;
            checkers[i] = new RowCheckerThread(bytes,start,end,pixelStride,width,height,colors,20,32);
            threads[i] = new Thread(checkers[i]);
            threads[i].start();
        }
        for(int i = 0; i < split;i++){
            try {
                threads[i].join();
            } catch (InterruptedException e){
                System.out.println(i);
                System.out.println(e);
            }
        }

        int masklen = 64;
        long[][][] results;
        HashMap<Integer, ArrayList<Blob>> blobsMap = new HashMap<>();
        ArrayList<Blob> blobs = new ArrayList<>();
        //Bugs in this loop
        //Row needs to take into account which split we are in for our HashMap
        //minsize and splitting might make some blobs be left out
        for(int i = 0; i < split;i++){
            results = checkers[i].getResults();
            int numColumns = results[0][0].length -1;
            int numColors = results.length;
            int numRows = results[0].length - 1;
            int lastVisitedRow;
            for(int color = 0; color < numColors;color++){
                for(int column = 0; column < numColumns; column++){
                    if(results[color][numRows][column] == 0 )
                        continue;
                    for(int row = 0; row < numRows;row++){
                        if(results[color][row][numColumns] == 0)
                            continue;
                        lastVisitedRow = row + 1;
                        long aproximateBlob = results[color][row][column];
                        while ((lastVisitedRow < (numRows - 1)) && ((results[color][lastVisitedRow+1][column] != 0))){
                            aproximateBlob |= results[color][lastVisitedRow][column];
                            lastVisitedRow++;
                        }
                        if(lastVisitedRow - row < minSize/2){
                            row = lastVisitedRow;
                            continue;
                        }
                        int left = 0;
                        int right = -1;
                        long mask = 0 | maskLeft;
                        //Depending on the ratio of minSize / maskLen we can have
                        //more than 1 partition, so it is not possible to do a more
                        //efficient check, will need to loop
                        for(int bitPosition = masklen; bitPosition >= 0;bitPosition--){
                            right++;
                            long current = aproximateBlob & mask;
                            if(current == 0){
                                if(right > left){
                                    int blob_start_y = row * pixels_h;
                                    int blob_start_x = (column * column_width) + left * pixels_w;
                                    int blob_end_y = blob_start_y + (lastVisitedRow - row) * pixels_h;
                                    int blob_end_x = (column * column_width) + right * pixels_w;
                                    Blob blob = new Blob();
                                    blob.setPosition(blob_start_x + (blob_end_x - blob_start_x)/2, blob_start_y + (blob_end_y - blob_start_y)/2);
                                    blob.setHeight(blob_end_y-blob_start_y);
                                    blob.setWidth(blob_end_x-blob_start_x);
                                    blob.setColorIndex(color);
                                    if(!blobsMap.containsKey(blob_start_x)){
                                        blobsMap.put(blob_start_x,new ArrayList<Blob>());
                                    }
                                    blobsMap.get(blob_start_x).add(blob);
                                }
                                left = right + 1;
                            }
                            mask >>>= 1;
                        }
                        row = lastVisitedRow;
                    }
                }
            }
        }

        Set<Integer> keys = blobsMap.keySet();
        List<Integer> keysList = keys.stream().collect(Collectors.toList());
        Collections.sort(keysList);
        for(Integer k: keysList){
            ArrayList<Blob> list = blobsMap.get(k);
            for(Blob element: list){
                int lookup = element.getPosition()[0] + element.getWidth()/2;
                if(!blobsMap.containsKey(lookup))
                    continue;
                int match = -1;
                ArrayList<Blob> foundList = blobsMap.get(lookup);
                for(int index = 0; index < foundList.size();index++){
                    if(element.getColorIndex() != foundList.get(index).getColorIndex())
                        continue;
                    int current_y_start = element.getPosition()[1] - element.getHeight()/2;
                    int current_y_end = element.getPosition()[1] + element.getHeight()/2;
                    int found_y_start = foundList.get(index).getPosition()[1] - foundList.get(index).getHeight()/2;
                    int found_y_end = foundList.get(index).getPosition()[1] - foundList.get(index).getHeight()/2;
                    if(found_y_start <= current_y_end && current_y_start <= found_y_end){
                        if(found_y_start < current_y_start)
                            current_y_start = found_y_start;
                        if(found_y_end > current_y_end)
                            current_y_end = found_y_end;
                        int current_x_start = element.getPosition()[0] - element.getWidth()/2;
                        int current_x_end = foundList.get(index).getPosition()[0] + foundList.get(index).getWidth()/2;
                        element.setWidth(current_x_end - current_x_start);
                        element.setHeight(current_y_end-current_y_start);
                        element.setPosition(current_x_start + (current_x_end - current_x_start)/2,current_y_start + (current_y_end-current_y_start)/2);
                        match = index;
                        break;
                    }
                }
                if(match != -1)
                    foundList.remove(match);
            }
        }
        for(Integer key:blobsMap.keySet()){
            for(Blob blob: blobsMap.get(key)){
                blobs.add(blob);
            }
        }
        return blobs;
    }

    class RowCheckerThread implements Runnable{
        long[][][] results;
        int[] bytes;
        int row_start;
        int row_end;
        int pixelStride;
        int width;
        int height;
        int[][] colors;
        int numColors;
        int minSize;
        int masklen = 64;
        int maxColorDistance = 20;
        int numElements;
        long colCount[][];

        public RowCheckerThread(int[] bytes, int row_start, int row_end, int pixelStride, int width, int height, int[][] colors, int maxColorDistance, int minSize) {
            this.bytes = bytes;
            this.row_start = row_start;
            this.row_end = row_end;
            this.pixelStride = pixelStride;
            this.width = width;
            this.height = height;
            this.colors = colors;
            this.maxColorDistance = maxColorDistance;
            this.minSize = minSize < masklen/2 ? masklen/2 : minSize;
            numColors  = colors.length;
            this.numElements = (int)Math.ceil((double)width/(double)this.masklen)/2;
            results = new long[numColors][(row_end-row_start)/2+1][numElements + 1];
            colCount = new long[numColors][numElements];
        }

        @Override
        public void run(){
            int counter = 0;
            long[] mask = new long[numColors];
            long clearMask = 0L;
            long prevClearMask = 0L;
            int prevCount = 0;
            long setter = 1L << 63;
            int match = -1;
            boolean change = false;
            int row = 0;
            int \_y;
            for(int y = width * row_start; y < (width * row_end)/2; y++){
                \_y = y*2;
                if(match == -1){
                    for(int i = 0; i < numColors; i++){
                        if(rgbArrayDistanceLessThanOrEqual(bytes,pixelStride,\_y,colors[i],maxColorDistance)){
                            match = i;
                            counter++;
                            mask[i] |= setter;
                            clearMask |= setter;
                        }
                    }
                } else{
                    if(rgbArrayDistanceLessThanOrEqual(bytes,pixelStride,\_y,colors[match],maxColorDistance)){
                        counter++;
                        mask[match] |= setter;
                        clearMask |= setter;
                    } else {
                        if(counter + prevCount < minSize/2){
                            mask[match] = 0;
                            if(change && prevClearMask != 0 && prevClearMask != 0){
                                int rowPos = \_y % width;
                                int mod = rowPos % masklen;
                                int currentColumn = ((rowPos - mod)/ masklen)/2;
                                if(currentColumn < 1)
                                    currentColumn = 1;
                                try{
                                    results[match][row][currentColumn - 1] ^= prevClearMask;
                                    if(results[match][row][currentColumn - 1] == 0){
                                        results[match][row][currentColumn] = 0;
                                        results[match][row][numElements]--;
                                        colCount[match][currentColumn-1]--;
                                    }
                                } catch (ArrayIndexOutOfBoundsException e){
                                    e.printStackTrace();
                                }
                                change = false;
                            }
                        }
                        prevClearMask = 0;
                        clearMask = 0;
                        prevCount = 0;
                        for(int i = 0; i < numColors; i++){
                            if(i == match)
                                continue;
                            if(rgbArrayDistanceLessThanOrEqual(bytes,pixelStride,\_y,colors[i],maxColorDistance)){
                                match = i;
                                counter++;
                                mask[i] |= setter;
                                clearMask |= setter;
                            }
                        }
                        match = -1;
                    }
                }
                setter >>>= 1;
                int next = \_y+2;
                int nextRow = next/2 % width;
                if( nextRow % masklen == 0){
                    setter = 1L << 63;
                    prevCount = counter;
                    counter = 0;

                    int pos = nextRow == 0 ? numElements - 1 : (nextRow/ masklen - 1) % numElements;

                    for(int i = 0; i < numColors; i++){
                        results[i][row][pos] |= mask[i];
                        if(mask[i] != 0){
                            results[i][row][numElements]++;
                            colCount[i][pos]++;
                        }
                        mask[i] = 0;
                    }
                    change = true;
                    if(match != -1){
                        prevClearMask = clearMask;
                        clearMask = 0L;
                    }
                }
                if(nextRow == 0){
                    if(match != -1 && prevCount < minSize/2){
                        results[match][row][numElements - 1] ^= prevClearMask;
                        if(results[match][row][numElements - 1] == 0){
                            results[match][row][numElements]--;
                            colCount[match][numElements -1]--;
                        }
                    }
                    counter = 0;
                    prevCount = 0;
                    match = -1;
                    clearMask = 0L;
                    prevClearMask = 0L;
                    row++;
                    change = false;
                    y += width/2;
                }
            }
            for(int color = 0; color<numColors;color++){
                for(int column = 0; column < numElements;column++){
                    results[color][(row_end-row_start)/2][column] = colCount[color][column];
                }
            }
        }
        public long[][][] getResults(){
            return results;
        }
    }

    private static boolean rgbArrayDistanceLessThanOrEqual(int[] bytes, int pixelStride, int position, int[] color, int distance){
        if(pixelStride == 1){
            int \_color = (color[0]<<16) | (color[1]<<8) | color[2];
            return (rgbDistance(bytes[position], \_color) <= distance);
        }
        int \_shift = pixelStride - 3;
        double rsq = Math.pow(bytes[position+\_shift] - color[\_shift], 2);
        double gsq = Math.pow(bytes[position+\_shift+1] - color[\_shift+1], 2);
        double bsq = Math.pow(bytes[position+\_shift+2] - color[\_shift+2], 2);
        return (Math.sqrt(rsq+gsq+bsq) <= distance);
    }
    private static double rgbDistance(int a, int b){
        int  red   = (a & 0x00ff0000) >> 16;
        int  green = (a & 0x0000ff00) >> 8;
        int  blue  =  a & 0x000000ff;
        int  red_2   = (b & 0x00ff0000) >> 16;
        int  green_2 = (b & 0x0000ff00) >> 8;
        int  blue_2  =  b & 0x000000ff;
        double rsq = Math.pow(red_2 - red, 2);
        double gsq = Math.pow(green_2-green,2);
        double bsq = Math.pow(blue_2-blue,2);
        return Math.sqrt(rsq+gsq+bsq);
    }
}
{% endhighlight %}

I went on to modify the Camera2 API example to test on my phone. I did not think that my phone's camera could disappoint me more (bad photo quality), but my code would give me an exception on my Honor 10. Searching on Google, wasn't very helpful. The issue was that even though I was requesting the image as ARGB_8888( from the Config enum in the Bitmap class), when reading from the ImageReader I was being told the format I got didn't match what I requested. Upon investigating a bit more, it seems that they are using some private format, but this is only conjecture. I decided to test on a Moto G5 Plus and everything seemed to work, more or less. Images from my camera weren't as perfect as the test images I quickly made using Krita. I haven't tested with leds yet and set my matching colors to perfect red, green and blue, but I may test changing my code to compare with the last match instead of the color from my list of colors.

I did a number of changes to the sample code, but roughly the important pieces are:

{% highlight java %}

  private void openCamera(int width, int height) {
        if (ContextCompat.checkSelfPermission(getActivity(), Manifest.permission.CAMERA)
                != PackageManager.PERMISSION_GRANTED) {
            requestCameraPermission();
            return;
        }
        setUpCameraOutputs(width, height);
        Activity activity = getActivity();
        CameraManager manager = (CameraManager) activity.getSystemService(Context.CAMERA_SERVICE);
        try {
            if (!mCameraOpenCloseLock.tryAcquire(2500, TimeUnit.MILLISECONDS)) {
                throw new RuntimeException("Time out waiting to lock camera opening.");
            }
            CameraCharacteristics characteristics =
                    manager.getCameraCharacteristics(mCameraId);
            StreamConfigurationMap map =
                    characteristics.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP);
            if (map == null) {
                Log.e(TAG, "No StreamConfigurationMap available");
                return;
            }
            Size[] sizes = map.getOutputSizes(Allocation.class);
            if (sizes.length == 0)
                return;
            Size videoSize = chooseOptimalSize(sizes, width, height, MAX_PREVIEW_WIDTH, MAX_PREVIEW_HEIGHT,sizes[0]);
            Size sz = sizes[sizes.length-2]; //Hardcoded size for testing different sizes
            mImageReader = ImageReader.newInstance(sz.getWidth(),
                    sz.getHeight(),
                    PixelFormat.RGBA_8888, 4);
            mImageReader.setOnImageAvailableListener(mmImageAvailable,mBackgroundHandler);
            mPreviewSize = videoSize;
            configureTransform(width, height);

            manager.openCamera(mCameraId, mStateCallback, mBackgroundHandler);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            throw new RuntimeException("Interrupted while trying to lock camera opening.", e);
        }
    }

        private final ImageReader.OnImageAvailableListener mmImageAvailable = new ImageReader.OnImageAvailableListener() {
        @Override
        public void onImageAvailable(ImageReader reader) {

            Image image = reader.acquireLatestImage();
            if (image == null)
                return;
            int width = image.getWidth();
            int height = image.getHeight();

            final Image.Plane[] planes = image.getPlanes();
            if(planes == null || planes.length == 0){
                ErrorDialog.newInstance(getString(R.string.camera_error))
                        .show(getChildFragmentManager(), FRAGMENT_DIALOG);
            }
            final ByteBuffer buffer = planes[0].getBuffer();
            int pixelStride = planes[0].getPixelStride();
            int rowStride = planes[0].getRowStride();
            int rowPadding = rowStride - pixelStride * width;

            Bitmap bitmap = Bitmap.createBitmap(width + rowPadding / pixelStride, height, Bitmap.Config.ARGB_8888);
            bitmap.copyPixelsFromBuffer(buffer);
            image.close();
            int[][] colors = { {255, 0, 0}, {0, 0, 255}, {0, 255, 0} };
            //The data in the byte array from the ByteBuffer was strange
            //feeling a bit lazy so I'll just get the array from the bitmap
            //I'll use it anyway to update the ImageView
            int[] arr = new int[width*height];
            bitmap.getPixels(arr,0,width,0,0,width,height);
            FastBlobFinderSkip finderSkip = new FastBlobFinderSkip(arr,width,height,1,colors);
            ArrayList<Blob> blobs = finderSkip.find(1);
            for(Blob blob: blobs){
                int[] pos = blob.getPosition();
                int w = blob.getWidth();
                int h = blob.getHeight();
                drawSquare(bitmap, pos[0], pos[1], w, h);
            }
            boolean findBlobs = false;
            updateImage(bitmap);
        }
    };

{% endhighlight %}

This is it for part one. For part two I'll move on to using an Arduino and testing this code in the conditions it is intended to be used.
