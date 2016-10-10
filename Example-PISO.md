module PISO<INT Size>
{
    public bit Clk;
    public bit Load;
    public bit[Size] In;
    public out bit Out => data[0];

    reg bit[Size] data;
    
    event on(Load) data = Load;
    event on(Clk) data >>= 1;
}
