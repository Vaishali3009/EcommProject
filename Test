import com.rbs.bdd.infrastructure.config.SoapLoggingInterceptor;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.ws.context.MessageContext;
import org.springframework.ws.soap.SoapMessage;
import org.springframework.ws.WebServiceMessage;

import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.Mockito.*;

class SoapLoggingInterceptorTest {

    private SoapLoggingInterceptor interceptor;
    private MessageContext context;

    @BeforeEach
    void setup() {
        interceptor = new SoapLoggingInterceptor();
        context = mock(MessageContext.class);

        WebServiceMessage mockMessage = mock(SoapMessage.class);
        when(context.getRequest()).thenReturn(mockMessage);
        when(context.getResponse()).thenReturn(mockMessage);
    }

    @Test
    void testHandleRequest() throws Exception {
        boolean result = interceptor.handleRequest(context, null);
        assertTrue(result);
    }

    @Test
    void testHandleResponse() throws Exception {
        boolean result = interceptor.handleResponse(context, null);
        assertTrue(result);
    }

    @Test
    void testHandleFault() throws Exception {
        boolean result = interceptor.handleFault(context, null);
        assertTrue(result);
    }

    @Test
    void testAfterCompletion() throws Exception {
        interceptor.afterCompletion(context, null, null);
        // no assertion required, just verifying no exceptions
    }
}
